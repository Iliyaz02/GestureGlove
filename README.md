# GestureGlove
# Gesture Glove Code: Sign Language to Text and Speech

import time
import cv2

# Try to import required modules, handle if missing
try:
    import mediapipe as mp
except ImportError:
    print("Warning: mediapipe module not found. Gesture recognition will not work.")
    mp = None

try:
    import pyttsx3
except ImportError:
    print("Warning: pyttsx3 module not found. Text-to-speech will be disabled.")
    pyttsx3 = None

try:
    import smbus
except ImportError:
    print("Warning: smbus module not found. I2C LCD functionality will be disabled.")
    smbus = None

# Initialize MediaPipe if available
if mp:
    try:
        mp_hands = mp.solutions.hands
        mp_drawing = mp.solutions.drawing_utils
    except AttributeError:
        print("Error initializing MediaPipe hands solutions.")
        mp_hands = None
        mp_drawing = None
else:
    mp_hands = None
    mp_drawing = None

# Initialize text-to-speech engine if available
if pyttsx3:
    try:
        engine = pyttsx3.init()
    except Exception as e:
        print(f"Error initializing pyttsx3: {e}")
        engine = None
else:
    engine = None

# Initialize I2C LCD if smbus available
if smbus:
    I2C_ADDR = 0x27
    LCD_WIDTH = 16
    try:
        bus = smbus.SMBus(1)
    except Exception as e:
        print(f"Error initializing SMBus: {e}")
        bus = None
else:
    bus = None

# LCD helper functions
def lcd_init():
    if not bus:
        print("LCD initialization skipped (smbus not available).")
        return
    try:
        bus.write_byte_data(I2C_ADDR, 0x00, 0x33)
        bus.write_byte_data(I2C_ADDR, 0x00, 0x32)
        bus.write_byte_data(I2C_ADDR, 0x00, 0x06)
        bus.write_byte_data(I2C_ADDR, 0x00, 0x0C)
        bus.write_byte_data(I2C_ADDR, 0x00, 0x28)
        bus.write_byte_data(I2C_ADDR, 0x00, 0x01)
        time.sleep(0.05)
    except Exception as e:
        print(f"LCD Initialization Error: {e}")

def lcd_message(message):
    if not bus:
        print("LCD message skipped (smbus not available).")
        return
    try:
        bus.write_byte_data(I2C_ADDR, 0x00, 0x80)  # Set cursor position
        for char in message.ljust(LCD_WIDTH):
            bus.write_byte_data(I2C_ADDR, 0x40, ord(char))
    except Exception as e:
        print(f"LCD Message Error: {e}")

# Initialize LCD (optional)
lcd_init()

# Gesture dictionary mapping recognized gestures to text
gesture_dict = {
    "thumbs_up": "OK",
    "fist": "Hello",
    "open_hand": "Stop"
}

# Camera setup
cap = cv2.VideoCapture(0)

if mp_hands:
    with mp_hands.Hands(min_detection_confidence=0.7, min_tracking_confidence=0.7) as hands:
        while cap.isOpened():
            success, image = cap.read()
            if not success:
                print("Ignoring empty camera frame.")
                continue

            # Flip image for mirror effect and convert color
            image = cv2.flip(image, 1)
            image_rgb = cv2.cvtColor(image, cv2.COLOR_BGR2RGB)

            # Process image and find hands
            result = hands.process(image_rgb)

            if result.multi_hand_landmarks:
                for hand_landmarks in result.multi_hand_landmarks:
                    # Draw hand landmarks on image
                    mp_drawing.draw_landmarks(image, hand_landmarks, mp_hands.HAND_CONNECTIONS)

                    # TODO: Replace this placeholder with actual gesture recognition logic
                    gesture = "thumbs_up"  # Example placeholder gesture

                    if gesture in gesture_dict:
                        text = gesture_dict[gesture]
                        lcd_message(text)
                        if engine:
                            engine.say(text)
                            engine.runAndWait()
                        print(f"Gesture recognized: {text}")

            cv2.imshow("Gesture Recognition", image)

            # Exit loop on 'q' key press
            if cv2.waitKey(5) & 0xFF == ord('q'):
                break
else:
    print("Mediapipe hands solution unavailable. Cannot run gesture recognition.")

# Release camera and close windows
cap.release()
cv2.destroyAllWindows()
