# Recording-the-servo-arm
in this project i control the servo arm using potentiometer and save the record and play it in the loop 
hare Button 14, Starts the record and stops the record 
Button 13 Saves the Record 
Button 12 plays the Record in a loop
Button 11 Resets the Record to record the new record 
COAD #
import machine
import utime

# Buttons
button_record = machine.Pin(14, machine.Pin.IN, machine.Pin.PULL_UP)  # Start/End recording
button_save = machine.Pin(13, machine.Pin.IN, machine.Pin.PULL_UP)    # Save sequence
button_play = machine.Pin(12, machine.Pin.IN, machine.Pin.PULL_UP)    # Play in loop
button_reset = machine.Pin(11, machine.Pin.IN, machine.Pin.PULL_UP)   # Reset everything

# Servo
servo = machine.PWM(machine.Pin(15))
servo.freq(50)

# Potentiometer
pot = machine.ADC(26)

# State variables
sequence = []
recording = False
playing = False

last_record = 1
last_save = 1
last_play = 1
last_reset = 1

last_time = utime.ticks_ms()
play_index = 0
step_start = 0

# Convert angle to PWM
def set_servo(angle):
    duty = int(1638 + (angle / 180) * 6553)
    servo.duty_u16(duty)

# Reset system
def reset_system():
    global sequence, recording, playing, play_index
    print("System Reset")
    sequence = []
    recording = False
    playing = False
    play_index = 0
    set_servo(90)  # Center servo

# Main loop
while True:
    now = utime.ticks_ms()

    r = button_record.value()
    save = button_save.value()
    play = button_play.value()
    reset = button_reset.value()

    # Reset button
    if reset == 0 and last_reset == 1:
        reset_system()

    # Read potentiometer
    pot_value = pot.read_u16()
    angle = int((pot_value / 65535) * 180)

    # Toggle recording with GP14
    if r == 0 and last_record == 1:
        recording = not recording
        if recording:
            print("Recording started")
            sequence = []        # Clear previous sequence
            last_time = utime.ticks_ms()
        else:
            print("Recording stopped")

    # While recording, save positions
    if recording:
        set_servo(angle)
        # Record each step when GP14 is pressed during recording
        duration = utime.ticks_diff(now, last_time)
        if sequence:
            sequence[-1] = (sequence[-1][0], duration)
        sequence.append((angle, 0))
        last_time = now

    # Save button (GP13)
    if save == 0 and last_save == 1 and sequence:
        duration = utime.ticks_diff(now, last_time)
        if sequence:
            sequence[-1] = (sequence[-1][0], duration)
        print("Sequence saved:", sequence)
        recording = False
        playing = False
        play_index = 0
        step_start = utime.ticks_ms()

    # Play button (GP12)
    if play == 0 and last_play == 1 and sequence:
        print("Playback started")
        playing = True
        play_index = 0
        step_start = now
        set_servo(sequence[0][0])

    # Playback engine (loop)
    if playing and sequence:
        angle_step, duration = sequence[play_index]
        if utime.ticks_diff(now, step_start) >= duration:
            play_index += 1
            if play_index >= len(sequence):
                play_index = 0
            angle_step, duration = sequence[play_index]
            set_servo(angle_step)
            step_start = now

    # Update last states
    last_record = r
    last_save = save
    last_play = play
    last_reset = reset

    utime.sleep_ms(20)
