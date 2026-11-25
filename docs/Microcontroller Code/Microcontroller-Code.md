---
title: Microcontroller Code
---

## Overview

This is the code used for the Curiosity Nano Microcontroller. Please refer [*here*](EGR304FinalProject.zip) for the ZIP file of the entire project.

## Code

```c
#include "mcc_generated_files/system/system.h"
#include "mcc_generated_files/dac/dac1.h"
#include "mcc_generated_files/uart/uart1.h"
#include "mcc_generated_files/uart/uart5.h"

#include <stdint.h>
#include <stdbool.h>

// Needed for __delay_us / __delay_ms.
// SET THIS TO YOUR ACTUAL CLOCK (Hz) if MCC has not already defined it.
#ifndef _XTAL_FREQ
#define _XTAL_FREQ          64000000UL
#endif

// AUDIO CONFIGURATION
#define AUDIO_MIN           1U
#define AUDIO_MAX           100U
#define DAC_MAX             255U
#define DAC_MIDPOINT        128U

// CONTROL CONFIGURATION
#define CONTROL_THRESHOLD   50U

// UART5 TRANSMISSION STATES
#define TX_IDLE             0U
#define TX_ON               1U
#define TX_OFF              2U

// BEEP / REPEAT MODES
#define BEEP_NONE           0U      // no beeps: DAC holds the last UART1 level
#define BEEP_HIGH           1U      // value >= threshold: 3 short high beeps
#define BEEP_LOW            2U      // value <  threshold: 1 long low beep

// Silence between repeats of the beep sequence
#define REPEAT_GAP_MS       400U

// AUDIO VARIABLES
static uint8_t audioSample;
static uint8_t dacValue;
static uint8_t lastDacValue = DAC_MIDPOINT;   // last level received on UART1

// CONTROL VARIABLES
static uint8_t controlSample;
static bool controlInitialized = false;
static bool controlIsOn = false;

// ASCII CONTROL INPUT BUFFER
static char controlBuffer[4];
static uint8_t controlIndex = 0;
static bool controlInvalid = false;

// UART5 TRANSMISSION VARIABLES
static char txBuffer[6];
static uint8_t txLength = 0;
static uint8_t txIndex = 0;
static uint8_t pendingMessage = TX_IDLE;
static uint8_t activeMessage = TX_IDLE;

// UART5 ECHO QUEUE
#define ECHO_SIZE           16U
static char echoBuffer[ECHO_SIZE];
static uint8_t echoHead = 0;
static uint8_t echoTail = 0;

// REPEAT VARIABLE
// Set by a valid UART5 entry. Cleared as soon as ANY new byte arrives
// on UART1 or UART5.
static uint8_t repeatMode = BEEP_NONE;

// One cycle of a sine wave, 16 samples, scaled to 0-255
static const uint8_t sineTable[16] =
{
    128, 176, 218, 246, 255, 246, 218, 176,
    128,  80,  38,  10,   1,  10,  38,  80
};


// ----------------------------------------
// AUDIO SAMPLE CONVERSION
// ----------------------------------------

static uint8_t ConvertSample(uint8_t sample)
{
    return (uint8_t)(
        ((uint16_t)(sample - AUDIO_MIN) * DAC_MAX) /
        (AUDIO_MAX - AUDIO_MIN)
    );
}


// ----------------------------------------
// PUSHBUTTON AND LED TEST
// RC1: external pull-up, button pulls LOW when pressed
// RF4: LED is active HIGH
// ----------------------------------------

static void TestButtonAndLED(void)
{
    if (PORTCbits.RC1 == 0)
    {
        LATFbits.LATF4 = 1;   // pressed -> LED ON
    }
    else
    {
        LATFbits.LATF4 = 0;   // released -> LED OFF
    }
}


// ----------------------------------------
// CONTROL MESSAGE QUEUE
// ----------------------------------------

static void QueueControlMessage(uint8_t message)
{
    pendingMessage = message;
}


// ----------------------------------------
// ECHO QUEUE
// ----------------------------------------

static void QueueEcho(char c)
{
    uint8_t next = (uint8_t)((echoHead + 1U) % ECHO_SIZE);

    if (next == echoTail)
        return;                     // queue full, drop echo

    echoBuffer[echoHead] = c;
    echoHead = next;
}


// ----------------------------------------
// NEW INPUT CHECK
// True if a byte is waiting on UART1 or UART5 (not consumed here).
// Used to cut a beep short so the new input is handled right away.
// ----------------------------------------

static bool NewInputPending(void)
{
    return UART1_IsRxReady() || UART5_IsRxReady();
}


// ----------------------------------------
// BEEP GENERATION
// Pitch is set by the delay between DAC samples.
// Approximate pitch: high ~1 kHz, low ~500 Hz.
// Code overhead adds a little time per sample, so the real pitch
// comes out slightly lower. Adjust the __delay_us values to tune.
//
// These functions return false if they stopped early because
// new input arrived.
// ----------------------------------------

static bool PlayTone(bool high, uint16_t cycles)
{
    for (uint16_t c = 0; c < cycles; c++)
    {
        if (NewInputPending())
            return false;

        for (uint8_t s = 0; s < 16U; s++)
        {
            DAC1_SetOutput(sineTable[s]);

            if (high)
            {
                __delay_us(50);
            }
            else
            {
                __delay_us(110);
            }
        }
    }

    DAC1_SetOutput(DAC_MIDPOINT);
    return true;
}

static bool SilenceMs(uint16_t ms)
{
    for (uint16_t i = 0; i < ms; i++)
    {
        if (NewInputPending())
            return false;

        __delay_ms(1);
    }

    return true;
}

static bool PlayBeepSequence(uint8_t mode)
{
    if (mode == BEEP_HIGH)
    {
        // 3 short high beeps (~100 ms each, ~80 ms gaps)
        for (uint8_t n = 0; n < 3U; n++)
        {
            if (!PlayTone(true, 100U))
                return false;

            if (!SilenceMs(80U))
                return false;
        }
    }
    else if (mode == BEEP_LOW)
    {
        // 1 long low beep (~300 ms)
        if (!PlayTone(false, 150U))
            return false;

        if (!SilenceMs(80U))
            return false;
    }

    return true;
}

static bool UART5TxIsIdle(void)
{
    return (txIndex >= txLength) &&
           (echoTail == echoHead) &&
           (pendingMessage == TX_IDLE);
}

// Repeats the last UART5 beep sequence until new input arrives on
// UART1 or UART5. Waits until all UART5 text has been sent first, so
// the echo and ON/OFF reply appear in PuTTY before the sound.
static void ProcessBeep(void)
{
    if (repeatMode == BEEP_NONE)
        return;

    if (!UART5TxIsIdle())
        return;

    bool completed = PlayBeepSequence(repeatMode);

    if (completed)
    {
        completed = SilenceMs(REPEAT_GAP_MS);
    }

    if (!completed)
    {
        // New input arrived mid-sound: restore the last UART1 level.
        // The input itself is read on the next pass of the main loop,
        // which also stops the repeat.
        DAC1_SetOutput(lastDacValue);
    }
}


// ----------------------------------------
// CONTROL VALUE PROCESSING
// ----------------------------------------

static void ProcessControlValue(uint8_t value)
{
    if (value < AUDIO_MIN || value > AUDIO_MAX)
        return;

    controlSample = value;

    bool newState = (controlSample >= CONTROL_THRESHOLD);

    // Every valid entry starts a repeating beep sequence
    repeatMode = newState ? BEEP_HIGH : BEEP_LOW;

    // ON/OFF text is only sent when the state changes
    if (!controlInitialized || newState != controlIsOn)
    {
        controlIsOn = newState;
        controlInitialized = true;

        if (controlIsOn)
        {
            QueueControlMessage(TX_ON);
        }
        else
        {
            QueueControlMessage(TX_OFF);
        }
    }
}


// ----------------------------------------
// UART5 ASCII INPUT (WITH ECHO)
// Accepts numbers 1-100 followed by Enter
// Backspace/Delete removes the last digit
// ----------------------------------------

static void ProcessControlInput(void)
{
    static char lastChar = 0;

    if (!UART5_IsRxReady())
        return;

    char received = UART5_Read();

    // New input stops whatever was repeating
    // (a valid entry below starts a new one)
    repeatMode = BEEP_NONE;

    // End of input line (echo CR/LF once, even if terminal sends CR+LF)
    if (received == '\r' || received == '\n')
    {
        if (!(received == '\n' && lastChar == '\r'))
        {
            QueueEcho('\r');
            QueueEcho('\n');
        }

        if (controlIndex > 0 && !controlInvalid)
        {
            uint16_t value = 0;

            for (uint8_t i = 0; i < controlIndex; i++)
            {
                value = (value * 10U) + (controlBuffer[i] - '0');
            }

            if (value >= AUDIO_MIN && value <= AUDIO_MAX)
            {
                ProcessControlValue((uint8_t)value);
            }
        }

        controlIndex = 0;
        controlInvalid = false;
    }

    // Backspace / Delete
    else if (received == '\b' || received == 0x7F)
    {
        if (controlIndex > 0 && !controlInvalid)
        {
            controlIndex--;
            QueueEcho('\b');
            QueueEcho(' ');
            QueueEcho('\b');
        }
    }

    // Accept numeric characters
    else if (received >= '0' && received <= '9')
    {
        if (controlIndex < 3 && !controlInvalid)
        {
            controlBuffer[controlIndex++] = received;
            QueueEcho(received);
        }
        else
        {
            controlInvalid = true;
        }
    }

    // Reject other characters
    else
    {
        controlInvalid = true;
    }

    lastChar = received;
}


// ----------------------------------------
// UART5 TRANSMISSION
// Priority: finish ON/OFF in progress, then echo, then load new ON/OFF
// ----------------------------------------

static void ProcessUART5(void)
{
    if (!UART5_IsTxReady())
        return;

    // 1. Finish an ON/OFF message already in progress (never interleave)
    if (txIndex < txLength)
    {
        UART5_Write(txBuffer[txIndex++]);
        return;
    }

    // 2. Send queued echo characters
    if (echoTail != echoHead)
    {
        UART5_Write(echoBuffer[echoTail]);
        echoTail = (uint8_t)((echoTail + 1U) % ECHO_SIZE);
        return;
    }

    // 3. Load the next pending ON/OFF message
    if (pendingMessage != TX_IDLE)
    {
        activeMessage = pendingMessage;
        pendingMessage = TX_IDLE;

        if (activeMessage == TX_ON)
        {
            txBuffer[0] = 'O';
            txBuffer[1] = 'N';
            txBuffer[2] = '\r';
            txBuffer[3] = '\n';
            txLength = 4;
        }
        else
        {
            txBuffer[0] = 'O';
            txBuffer[1] = 'F';
            txBuffer[2] = 'F';
            txBuffer[3] = '\r';
            txBuffer[4] = '\n';
            txLength = 5;
        }

        txIndex = 0;
    }
}


// ----------------------------------------
// AUDIO INPUT: UART1 RX / RC3
// The DAC holds the last valid level until new data arrives.
// Any byte received (valid or not) stops a repeating beep.
// ----------------------------------------

static void ProcessAudioInput(void)
{
    if (UART1_IsRxReady())
    {
        audioSample = UART1_Read();

        // New input stops whatever was repeating
        repeatMode = BEEP_NONE;

        if (audioSample >= AUDIO_MIN &&
            audioSample <= AUDIO_MAX)
        {
            dacValue = ConvertSample(audioSample);
            lastDacValue = dacValue;
        }

        // Output the last valid level (unchanged if the byte was ignored)
        DAC1_SetOutput(lastDacValue);
    }
}


// ----------------------------------------
// MAIN PROGRAM
// ----------------------------------------

int main(void)
{
    SYSTEM_Initialize();

    // RC1: Pushbutton input
    ANSELCbits.ANSELC1 = 0;
    TRISCbits.TRISC1 = 1;

    // RF4: LED output
    ANSELFbits.ANSELF4 = 0;
    TRISFbits.TRISF4 = 0;

    // LED initially OFF (active HIGH)
    LATFbits.LATF4 = 0;

    // Initialize DAC midpoint
    DAC1_SetOutput(DAC_MIDPOINT);

    while (1)
    {
        // Pushbutton and LED test
        TestButtonAndLED();

        // Receive audio samples
        ProcessAudioInput();

        // Receive control numbers
        ProcessControlInput();

        // Transmit echo and ON/OFF messages
        ProcessUART5();

        // Repeat the last beep sequence until new input arrives
        ProcessBeep();
    }
}
