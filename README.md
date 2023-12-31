# Coding3 Project :exploding_head:
## Statement
For this project, libraries used are mediapipe and opencv, and also bit of numpy. You can draw without mouse or keyboard.
I have used mediapipe's solution for hand landmark detection.
It detects 21 different keypoints. And you can draw the paint with 3 keypoints of them.
This project will capture your hand's movement, and you can draw on the air canvas with your hand as a pen.

Here is demo video: https://youtu.be/P6OClikTveI
## Coding Part
### import everything
```
import mediapipe as mp
import cv2
import numpy as np
import time
```

### Constants
```
ml = 150  # Left margin for the tools area
max_x, max_y = 250 + ml, 50  // Maximum coordinates for tools area
curr_tool = "select tool"  // Current selected tool
time_init = True  // Flag to track tool selection time
rad = 40  // Initial radius for tool selection circle*
var_inits = False  // Flag to track tool-specific variables initialization
thick = 4  // Thickness of drawing lines/shapes
prevx, prevy = 0, 0  // Previous x, y coordinates
```

### Get tools function
```
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
```

### Initialize Mediapipe hands module
```
hands = mp.solutions.hands
hand_landmark = hands.Hands(
    min_detection_confidence=0.6, min_tracking_confidence=0.6, max_num_hands=1
)
draw = mp.solutions.drawing_utils
```

### Load tools image
```
tools = cv2.imread("tools.png")
tools = tools.astype("uint8")
```

### Create a mask for drawing
```
mask = np.ones((480, 640)) * 255
mask = mask.astype("uint8")
```

### Open camera capture
```
cap = cv2.VideoCapture(0)
while True:
    _, frm = cap.read()
    frm = cv2.flip(frm, 1)
```
### Convert frame to RGB for Mediapipe
```    rgb = cv2.cvtColor(frm, cv2.COLOR_BGR2RGB)

    # Process hand landmarks using Mediapipe
    op = hand_landmark.process(rgb) // Process hand landmarks using Mediapipe

    if op.multi_hand_landmarks:  // Check if hand landmarks are detected
        for i in op.multi_hand_landmarks:
            # Draw hand landmarks on the frame
            draw.draw_landmarks(frm, i, hands.HAND_CONNECTIONS)
            x, y = int(i.landmark[8].x*640), int(i.landmark[8].y*480)  // Get coordinates of the tip of the index finger
```
### Check if hand is inside the tools area
```
    if x < max_x and y < max_y and x > ml:  // Check if the tip of the finger is within the tools area
            if time_init:
                ctime = time.time()  // Record the current time for tool selection timing
                time_init = False
            ptime = time.time()  // Get the current time for tool selection timing

```
### Draw selection circle on the frame
```
                cv2.circle(frm, (x, y), rad, (0, 255, 255), 2)
                rad -= 1

           // Check if enough time has passed for tool selection
                if (ptime - ctime) > 0.8:  // Check if enough time has passed for tool selection
                      curr_tool = getTool(x)  // Get the current selected tool based on the finger position
                      print("Your current tool is set to:", curr_tool)
                      time_init = True  // Reset the tool selection timing
                      rad = 40  # Reset the radius of the selection circle
            else:
                time_init = True
                rad = 40
```
#### 'Draw' section
```            
          // If the current tool is draw
           if curr_tool == "draw":
           // Get index finger and thumb tip coordinates
                xi, yi = int(i.landmark[12].x * 640), int(i.landmark[12].y * 480)
                y9 = int(i.landmark[9].y * 480)


           // Check if index finger is raised

                if index_raised(yi, y9): //Check if the index finger is raised
           // Draw line on the mask  
					cv2.line(mask, (prevx, prevy), (x, y), 0, thick)
					prevx, prevy = x, y

				else:
					prevx = x
					prevy = y
```
#### 'Line' section
```
           // If the current tool is line
			elif curr_tool == "line":
				xi, yi = int(i.landmark[12].x*640), int(i.landmark[12].y*480)   // Get the coordinates of finger landmarks
				y9  = int(i.landmark[9].y*480)

				if index_raised(yi, y9):
					if not(var_inits):
						xii, yii = x, y
						var_inits = True

					cv2.line(frm, (xii, yii), (x, y), (50,152,255), thick)

				else:
					if var_inits:
						cv2.line(mask, (xii, yii), (x, y), 0, thick)
						var_inits = False
```
#### 'Rectangle' section
```
           // If the current tool is rectangle
			elif curr_tool == "rectangle":
				xi, yi = int(i.landmark[12].x*640), int(i.landmark[12].y*480)   // Get the coordinates of finger landmarks
				y9  = int(i.landmark[9].y*480)

				if index_raised(yi, y9):
					if not(var_inits):
						xii, yii = x, y
						var_inits = True

					cv2.rectangle(frm, (xii, yii), (x, y), (0,255,255), thick)

				else:
					if var_inits:
						cv2.rectangle(mask, (xii, yii), (x, y), 0, thick)
						var_inits = False
```
#### 'Circle' section
```
           // If the current tool is circle
			elif curr_tool == "circle":
				xi, yi = int(i.landmark[12].x*640), int(i.landmark[12].y*480)   // Get the coordinates of finger landmarks
				y9  = int(i.landmark[9].y*480)

				if index_raised(yi, y9):
					if not(var_inits):
						xii, yii = x, y
						var_inits = True

					cv2.circle(frm, (xii, yii), int(((xii-x)**2 + (yii-y)**2)**0.5), (255,255,0), thick)

				else:
					if var_inits:
						cv2.circle(mask, (xii, yii), int(((xii-x)**2 + (yii-y)**2)**0.5), (0,255,0), thick)
						var_inits = False
```
#### 'Eraser' section
```
           // If the current tool is eraser
			elif curr_tool == "erase":
				xi, yi = int(i.landmark[12].x*640), int(i.landmark[12].y*480)     // Get the coordinates of finger landmarks
				y9  = int(i.landmark[9].y*480)

				if index_raised(yi, y9):
					cv2.circle(frm, (x, y), 30, (0,0,0), -1)
					cv2.circle(mask, (x, y), 30, 255, -1)


    //Draw a white circle on the mask (eraser)
	op = cv2.bitwise_and(frm, frm, mask=mask)
```
```
	frm[:, :, 1] = op[:, :, 1]
	frm[:, :, 2] = op[:, :, 2]

// Overlay the 'tools' image onto the frame in a specific region
	frm[:max_y, ml:max_x] = cv2.addWeighted(tools, 0.7, frm[:max_y, ml:max_x], 0.3, 0)

	cv2.putText(frm, curr_tool, (270+ml,30), cv2.FONT_HERSHEY_SIMPLEX, 1, (0,0,255), 2)
	cv2.imshow("paint app", frm)
    
// Wait for the user to press the Esc key (ASCII code 27)
	if cv2.waitKey(1) == 27:
		cv2.destroyAllWindows()
		cap.release()
		break
 
 ```
 
 
## Reference

Tutorial Video I have watched

Ivan Goncharov - Custom Hand Gesture Recognition with Hand Landmarks Using Google’s Mediapipe + OpenCV in Python
https://www.youtube.com/watch?v=a99p_fAr6e4

murtazasworkshop - AI Virtual Mouse | OpenCV Python | Computer Vision
https://www.youtube.com/watch?v=8gPONnGIPgw&t=11s
