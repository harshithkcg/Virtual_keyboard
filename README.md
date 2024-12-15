# Virtual_keyboard
import cv2
from cvzone.HandTrackingModule import HandDetector
import cvzone
from math import sqrt
from pynput.keyboard import Controller, Key
import tkinter as tk  # For displaying text in another window
import time  # Import time for the delay

# Initialize webcam and set resolution
cap = cv2.VideoCapture(0)
cap.set(3, 1280)  # Width
cap.set(4, 720)   # Height

# Initialize HandDetector
detector = HandDetector(detectionCon=0.8, maxHands=2)  # Allow detection of both hands

# Keyboard layout with "Space", "Backspace", "Clear", and "Caps" on the same row but with significant gaps
keys = [["Q", "W", "E", "R", "T", "Y", "U", "I", "O", "P"],
        ["A", "S", "D", "F", "G", "H", "J", "K", "L", ";"],
        ["Z", "X", "C", "V", "B", "N", "M", ",", ".", "/"],
        ["Space", "Caps", "Backspace", "Clear"]]  # Added Caps and adjusted Clear button position

finalText = ""
keyboard = Controller()

# Button class
class Button:
    def __init__(self, pos, text, size=[85, 85]):
        self.pos = pos
        self.size = size
        self.text = text
        self.is_pressed = False  # Track whether the button is pressed

# Draw buttons on the screen
def drawAll(img, buttonList):
    for button in buttonList:
        x, y = button.pos
        w, h = button.size

        # Change background color for pressed and unpressed buttons
        color = (0, 0, 255) if button.is_pressed else (234, 234, 234)  # Red for pressed, light gray for unpressed

        cvzone.cornerRect(img, (x, y, w, h), 20, rt=0)
        cv2.rectangle(img, button.pos, (x + w, y + h), color, cv2.FILLED)
        cv2.putText(img, button.text, (x + 20, y + 65) if button.text != "Backspace" else (x + 10, y + 50),
                    cv2.FONT_HERSHEY_PLAIN, 3, (0, 0, 0), 3)  # Black text for visibility
    return img

# Create buttons with adjusted placement of the Clear and Caps buttons
buttonList = []
x_offset = 50  # Starting X offset for buttons
y_offset = 50  # Starting Y offset for buttons

# Adjust keys layout to position Clear, Caps, Backspace, and Space accordingly
for i in range(len(keys)):
    for j, key in enumerate(keys[i]):
        size = [85, 85] if key not in ["Space", "Backspace", "Clear", "Caps"] else [150, 85]

        if key == "Space":
            buttonList.append(Button([x_offset + 500, y_offset + 100 * i], key, size))
        elif key == "Backspace":
            buttonList.append(Button([x_offset, y_offset + 100 * i], key, size))
        elif key == "Caps":
            buttonList.append(Button([x_offset + 650, y_offset + 100 * i], key, size))  # Increased distance
        elif key == "Clear":
            buttonList.append(Button([x_offset, 550], key, size))  # Adjusted for bottom-left position
        else:
            buttonList.append(Button([x_offset + j * 100, y_offset + 100 * i], key, size))

# Detect if two fingers are over a button
def checkKeyPress(hand, buttonList, finalText, img, last_key_press_time, key_press_delay):
    lmList = hand['lmList']  # Landmark list
    current_time = time.time()  # Get the current time

    for button in buttonList:
        x, y = button.pos
        w, h = button.size

        # Check if index finger (landmark 8) and middle finger (landmark 12) are over a button
        if x < lmList[8][0] < x + w and y < lmList[8][1] < y + h and x < lmList[12][0] < x + w and y < lmList[12][1] < y + h:
            # Highlight button in purple if hovered
            cv2.rectangle(img, (x - 5, y - 5), (x + w + 5, y + h + 5), (175, 0, 175), cv2.FILLED)
            cv2.putText(img, button.text, (x + 20, y + 65) if button.text != "Backspace" else (x + 10, y + 50),
                        cv2.FONT_HERSHEY_PLAIN, 3, (255, 255, 255), 3)

            # Calculate the distance between the index finger (landmark 8) and middle finger (landmark 12)
            x1, y1 = lmList[8][0], lmList[8][1]  # Index finger tip
            x2, y2 = lmList[12][0], lmList[12][1]  # Middle finger tip
            distance = sqrt((x2 - x1) ** 2 + (y2 - y1) ** 2)

            if distance < 30 and (current_time - last_key_press_time) > key_press_delay:  # Check delay
                if button.text == "Space":
                    keyboard.press(" ")
                    finalText += " "
                elif button.text == "Caps":  # Handle Caps key
                    finalText = finalText.upper() if finalText.islower() else finalText.lower()
                elif button.text == "Backspace":  # Handle Backspace key
                    keyboard.press(Key.backspace)
                    finalText = finalText[:-1] if finalText else ""
                elif button.text == "Clear":  # Handle Clear button
                    finalText = ""  # Clear the text in the Tkinter window
                else:
                    keyboard.press(button.text)
                    finalText += button.text

                # Update the button's pressed state and update the last key press time
                button.is_pressed = True
                last_key_press_time = current_time

                # Reset the button's pressed state after a short delay (visual feedback)
                time.sleep(0.2)
                button.is_pressed = False

    return finalText, last_key_press_time

# Create Tkinter window to display typed text
def createTextWindow():
    window = tk.Tk()
    window.title("Typed Text")
    window.geometry("500x400")
    text_widget = tk.Label(window, text="", font=("Arial", 20), width=30, height=10, anchor="nw", justify="left")
    text_widget.pack(padx=20, pady=20)
    return window, text_widget

# Initialize Tkinter window
text_window, text_label = createTextWindow()

# Set the initial values for last_key_press_time and key_press_delay
last_key_press_time = 0
key_press_delay = 1  # 1 second delay between key presses

# Main loop
while True:
    success, img = cap.read()
    img = cv2.flip(img, 1)  # Flip the image horizontally to create a mirror effect
    hands, img = detector.findHands(img)  # Detect hands and get annotated image
    img = drawAll(img, buttonList)       # Draw buttons on the image

    if hands:
        for hand in hands:  # Process each detected hand
            finalText, last_key_press_time = checkKeyPress(hand, buttonList, finalText, img, last_key_press_time, key_press_delay)

    # Display typed text in the Tkinter window
    text_label.config(text=finalText)

    # Show the image
    cv2.imshow("Image", img)

    # Update the Tkinter window
    text_window.update()

    if cv2.waitKey(1) & 0xFF == ord('q'):  # Press 'q' to quit
        break

cap.release()
cv2.destroyAllWindows()
text_window.quit()
