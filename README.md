# Robot Lego Spike Hub
Here you will find a program that will scan all ports on a Spike hub and set them up to do whatever you want. 
All the colors and hub codes are prepared for you including sound codes for instruments and songs.
<img style="width: 300px;" src="https://assets.education.lego.com/v3/assets/blt293eea581807678a/blt96801dfdfb5b7521/627118b9941a2939d3d00593/45678_prod_packaging_SPIKE_PRIME_Set_01.png?locale=en-us&auto=webp&format=jpeg&width=1600&quality=90&fit=bounds"/>
```
# import lego libraries
from hub import light_matrix, sound
import random, runloop, device, motor, time, motor_pair, color_sensor, force_sensor, color, distance_sensor
from app import music, sound as big_sound

# play pre-built sounds that play thru the laptop NOT the device
async def play_sounds(name='Dog Bark 1',volume=100,pitch=0,pan=0):
    await big_sound.play('Dog Bark 2')

# play sounds by note that play thru the Spike Hub
def play_notes(melody):
    try:
        frequency = duration = volume_level = 0# initialize note vars
        # Play melody
        for note in melody:
            frequency, duration, volume_level = note
            # Play each note with the given volume
            sound.beep(frequency, duration, volume_level, transition=0)
            # Add a small delay between notes for better separation
            time.sleep(duration / 1000.0 + 0.05)
    except Exception as e:
        print("Error Playing Sounds!:", e)
        light_matrix.show_image(light_matrix.IMAGE_CONFUSED)
        # sound.beep(440, 500, 100)# Default beep
        return

# play a song by instrument and tone via the Hub
def play_music(song):
    '''
    DRUM_BASS = 2
    DRUM_BONGO = 13
    DRUM_CABASA = 15
    DRUM_CLAVES = 9
    DRUM_CLOSED_HI_HAT = 6
    DRUM_CONGA = 14
    DRUM_COWBELL = 11
    DRUM_CRASH_CYMBAL = 4
    DRUM_CUICA = 18
    DRUM_GUIRO = 16
    DRUM_HAND_CLAP = 8
    DRUM_OPEN_HI_HAT = 5
    DRUM_SIDE_STICK = 3
    DRUM_SNARE = 1
    DRUM_TAMBOURINE = 7
    DRUM_TRIANGLE = 12
    DRUM_VIBRASLAP = 17
    DRUM_WOOD_BLOCK = 10
    INSTRUMENT_BASS = 6
    INSTRUMENT_BASSOON = 14
    INSTRUMENT_CELLO = 8
    INSTRUMENT_CHOIR = 15
    INSTRUMENT_CLARINET = 10
    INSTRUMENT_ELECTRIC_GUITAR = 5
    INSTRUMENT_ELECTRIC_PIANO = 2
    INSTRUMENT_FLUTE = 12
    INSTRUMENT_GUITAR = 4
    INSTRUMENT_MARIMBA = 19
    INSTRUMENT_MUSIC_BOX = 17
    INSTRUMENT_ORGAN = 3
    INSTRUMENT_PIANO = 1
    INSTRUMENT_PIZZICATO = 7
    INSTRUMENT_SAXOPHONE = 11
    INSTRUMENT_STEEL_DRUM = 18
    INSTRUMENT_SYNTH_LEAD = 20
    INSTRUMENT_SYNTH_PAD = 21
    INSTRUMENT_TROMBONE = 9
    INSTRUMENT_VIBRAPHONE = 16
    INSTRUMENT_WOODEN_FLUTE = 13
    '''
    instrument = note = duration = 0# initialize note vars
    try:
        # Play the song
        for note in song:
            instrument, note, duration = note
            if instrument == 'drum':
                music.play_drum(note)
            else:
                music.play_instrument(instrument, note, duration)
    except Exception as e:
        print("Error Playing Music!:", e)
        light_matrix.write(e)
        sound.beep(440, 500, 100)# Default beep
        return

# random True/False generator
def coin_toss():
    return random.choice([True, False])

# ports to scan
myports = [0,1,2,3,4,5] # 1 hub TODO: 2 hub chained config

# spike devices
devices_init = {
    'distance': 62,# Assuming 'distance' sensor is on port 1
    'big_motor': 49,
    'motor': 48,
    'color': 61,
    'bump': 63
    # Add more device IDs and names as needed
}

# set initial values to -1
devices_map = {
    'distance': -1,# Assuming 'distance' sensor is on port 1
    'big_motor': -1,
    'motor': -1,
    'color': -1,
    'bump': -1
    # Add more device IDs and names as needed
}

#  initialize Port devices:
left_wheel=right_wheel=big_motor = bump = right_color_sensor = left_color_sensor = distance_port = big_motor_port = -1
drivetrain = (left_wheel, right_wheel)

# Define a dictionary that maps color codes (integers) to messages
color_messages = {
    0: "Black detected",
    1: "Magenta detected",
    2: "Mystery? detected",
    3: "Blue detected",
    4: "Azure detected",
    5: "Orange detected",
    6: "Green detected",
    7: "Yellow detected",
    8: "Mystery? detected",
    9: "Red detected",
    10: "White detected"
}

# Define a list of colors (assuming the order matters)
colors = [
    "Black",
    "Magenta",
    "Mystery?",
    "Blue",
    "Azure",
    "Orange",
    "Green",
    "Yellow",
    "Red",
    "White"
]

# Set any motors or arms where you need them to start
async def calibration():
    global big_motor
    motor.reset_relative_position(big_motor,0)
    try:
        while motor.relative_position(big_motor) > -63:
            # print(motor.relative_position(big_motor))
            motor.run(big_motor, -100)
        motor.stop(big_motor)
    except Exception as e:
        print(e)
    return

# move the side wheels (#48) in tandem using motor pairing
async def myMove(time=1, steering=100, vel=100, accel=100):
    global drivetrain
    if time == 0:
        return
    if left_wheel != -1 and right_wheel != -1:
        drivetrain = (left_wheel, right_wheel)
        # print("Drivetrain initialized with left wheel: {}, right wheel: {}".format(left_wheel, right_wheel))
        motor_pair.pair(motor_pair.PAIR_1, drivetrain[1], drivetrain[0])
        motor_pair.move(motor_pair.PAIR_1, steering, velocity=vel, acceleration=accel)
    else:
        print("Drivetrain not fully initialized")
    await runloop.sleep_ms(time*1500)
    motor_pair.stop(motor_pair.PAIR_1)
    motor_pair.unpair(motor_pair.PAIR_1)
    return

# detect and assign each device by polling the ports
async def initialize_device(myport):
    global left_wheel, right_wheel, my_color_sensor, bump, left_color_sensor, right_color_sensor, distance_port, big_motor_port
    device_id = device.id(myport)# Assuming 'device.id' returns the device ID
    if device_id in devices_init.values():# Check if the device ID is in the values of the dictionary
        device_name = [key for key, value in devices_init.items() if value == device_id][0]
        if device_name == 'motor':
            if left_wheel == -1:
                left_wheel = myport
            else:
                right_wheel = myport
        elif device_name == 'big_motor':
            big_motor_port = myport
        elif device_name == 'color':
            if left_color_sensor == -1:
                left_color_sensor = myport
            else:
                right_color_sensor = myport
        elif device_name == 'bump':
            bump = myport
        elif device_name == 'distance':
            distance_port = myport
        print("Device {} is connected to port {}".format(device_name, myport))
    else:
        print("Unknown Device connected to port {}".format(myport))
    return

# scan the device ports that are ready and find the device
async def scan_devices():
    for myport in myports:
        if device.ready(myport):
            await initialize_device(myport)

# radar swiper to look for items in front
async def scan_environment(myport):
    global big_motor_port
    degrees = 0
    degrees_direction = 1 # 1 for Forward -1 for Reverse
    motor.reset_relative_position(big_motor_port,0)
    while distance_sensor.distance(myport) == -1:
        if degrees < 25000:
            degrees = degrees + 1
            motor.run_for_degrees(big_motor_port,(degrees_direction * degrees) * -1,50)
        else:
            degrees = 0
            await motor.run_to_relative_position(big_motor_port,0,600)
            runloop.sleep_ms(1000)
    sound.beep(770,1500,100)
    # Update all pixels on Distance Sensor using the show function
    # Create a list with 4 identical intensity values
    pixels = [100] * 4
    # Update all pixels to show same intensity
    distance_sensor.show(distance_port, pixels)
    await runloop.sleep_ms(1000)
    await motor.run_to_relative_position(big_motor_port,0,600)
    # print("EOR ", degrees * degrees_direction)
    await runloop.sleep_ms(1000)
    return((degrees * 9) * degrees_direction)

# here is where you put the cool stuff you want your robot to do
async def main():
    global bump, left_detected_color_code, right_detected_color_code, color_port
    # here is where you put whatever code you want to run the robot
        
runloop.run(main())
```
