# Ex3 - Intruder

&#x20;打開 `Terminal`

<figure><img src="../.gitbook/assets/terminal.png" alt=""><figcaption></figcaption></figure>

輸入以下指令，以安裝所須套件，安裝時間頗長，請耐心等候。

```sh
sudo apt update -y
sudo apt install -y python3-opencv libatlas-base-dev
pip3 install face_recognition
```

開新檔案 `registration.py`。 利用以下程式登記使用者。

{% code title="registration" lineNumbers="true" %}
```python
from picamera import PiCamera
from time import sleep
from os import makedirs, path

registration_directory = '/home/pi/registration'

if not path.exists(registration_directory):
    makedirs(registration_directory)
    
camera = PiCamera()

username = input('登記的使用者 (按Enter跳過)： ')

if username == '':
    exit()

while username != '':
    camera.start_preview()
    image_path = f'{registration_directory}/{username}.jpg'
    sleep(5)
    camera.capture(image_path)
    camera.stop_preview()

    username = input('下一個登記的使用者：(按Enter跳過) ')

camera.close()
print('登記完成')
```
{% endcode %}

開啟新檔 `intruder.py` ，將 `LED` 連接到 `GPIO5` 然後執行程式。

{% code title="intruder.py" lineNumbers="true" %}
```python
from os import listdir, path
from gpiozero import LED
import face_recognition
import cv2
import numpy as np

led = LED(5)

# 用作儲存已登記的 face_encoding 及 name
known_face_encodings = []
known_face_names = []

registration_directory = '/home/pi/registration'

# 如果未有 registration_directory, 將關閉程式
if not path.exists(registration_directory):
    exit()

# 從 registration_directory 中，載入並識別圖片
for image_file in listdir(registration_directory):
    filename, ext = path.splitext(image_file)
    if ext == '.jpg':
        image_location = path.join(registration_directory, image_file)
        user_image = face_recognition.load_image_file(image_location)
        face_encoding = face_recognition.face_encodings(user_image)[0]

    known_face_encodings.append(face_encoding)
    known_face_names.append(filename)

video_capture = cv2.VideoCapture(0)

face_locations = []
face_encodings = []
face_names = []
process_this_frame = True
UNKNOWN = "Unknown"

while True:
    ret, frame = video_capture.read()

    if process_this_frame:
        small_frame = cv2.resize(frame, (0, 0), fx=0.25, fy=0.25)
        code = cv2.COLOR_BGR2RGB
        rgb_small_frame = cv2.cvtColor(small_frame, code)

        face_locations = face_recognition.face_locations(rgb_small_frame)
        face_encodings = face_recognition.face_encodings(
            rgb_small_frame, face_locations
        )

        face_names = []
        tolerance = 0.5
        for face_encoding in face_encodings:
            matches = face_recognition.compare_faces(
                known_face_encodings, face_encoding, tolerance
            )

            face_distances = face_recognition.face_distance(
                known_face_encodings, face_encoding
            )

            best_match_index = np.argmin(face_distances)

            name = UNKNOWN

            if matches[best_match_index]:
                name = known_face_names[best_match_index]

            face_names.append(name)

    process_this_frame = not process_this_frame
    led.off()

    # 顯示結果
    for (top, right, bottom, left), name in zip(face_locations, face_names):
        top *= 4
        right *= 4
        bottom *= 4
        left *= 4

        # 畫上方框
        cv2.rectangle(frame, (left, top), (right, bottom), (0, 0, 255), 2)
        cv2.rectangle(
            frame,
            (left, bottom - 35),
            (right, bottom),
            (0, 0, 255),
            cv2.FILLED
        )

        # 寫上名稱
        font = cv2.FONT_HERSHEY_DUPLEX
        cv2.putText(
            frame,
            name,
            (left + 6, bottom - 6),
            font,
            1.0,
            (255, 255, 255),
            1
        )

        # 如果發現有未登錄的人，會亮起 LED
        try:
            face_names.index(UNKNOWN)
            led.on()
        except:
            pass

    cv2.imshow('Video', frame)

    # 按 'q' 關閉程式
    if cv2.waitKey(1) & 0xFF == ord('q'):
        break

video_capture.release()
cv2.destroyAllWindows()
```
{% endcode %}
