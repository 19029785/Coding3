# Coding3
#Import everything we need
import mediapipe as mp
import cv2
import numpy as np
import time

# Constants
ml = 150  # Left margin for the tools area
max_x, max_y = 250 + ml, 50  // Maximum coordinates for tools area
curr_tool = "select tool"  // Current selected tool
time_init = True  // Flag to track tool selection time
rad = 40  // Initial radius for tool selection circle
var_inits = False  // Flag to track tool-specific variables initialization
thick = 4  // Thickness of drawing lines/shapes
prevx, prevy = 0, 0  // Previous x, y coordinates

# Get tools function
def getTool(x):
    if x < 50 + ml:
        return "line"
    elif x < 100 + ml:
        return "rectangle"
    elif x < 150 + ml:
        return "draw"
    elif x < 200 + ml:
        return "circle"
    else:
        return "erase"

# Initialize Mediapipe hands module
hands = mp.solutions.hands
hand_landmark = hands.Hands(
    min_detection_confidence=0.6, min_tracking_confidence=0.6, max_num_hands=1
)
draw = mp.solutions.drawing_utils

# Load tools image
tools = cv2.imread("tools.png")
tools = tools.astype("uint8")

# Create a mask for drawing
mask = np.ones((480, 640)) * 255
mask = mask.astype("uint8")

# Open camera capture
cap = cv2.VideoCapture(0)
while True:
    _, frm = cap.read()
    frm = cv2.flip(frm, 1)

    # Convert frame to RGB for Mediapipe
    rgb = cv2.cvtColor(frm, cv2.COLOR_BGR2RGB)

    # Process hand landmarks using Mediapipe
    op = hand_landmark.process(rgb)

    if op.multi_hand_landmarks:
        for i in op.multi_hand_landmarks:
            # Draw hand landmarks on the frame
            draw.draw_landmarks(frm, i, hands.HAND_CONNECTIONS)
            x, y = int(i.landmark[8].x * 640), int(i.landmark[8].y * 480)

            # Check if hand is inside the tools area
            if x < max_x and y < max_y and x > ml:
                if time_init:
                    ctime = time.time()
                    time_init = False
                ptime = time.time()

                # Draw selection circle on the frame
                cv2.circle(frm, (x, y), rad, (0, 255, 255), 2)
                rad -= 1

                # Check if enough time has passed for tool selection
                if (ptime - ctime) > 0.8:
                    curr_tool = getTool(x)
                    print("Your current tool is set to:", curr_tool)
                    time_init = True
                    rad = 40
            else:
                time_init = True
                rad = 40

            if curr_tool == "draw":
                # Get index finger and thumb tip coordinates
                xi, yi = int(i.landmark[12].x * 640), int(i.landmark[12].y * 480)
                y9 = int(i.landmark[9].y * 480)

                # Check if index finger is raised
                if index_raised(yi, y9):
                    # Draw line on the mask
                    cv2.line(mask, (prevx, prevy), (x
