# Rural-smart-water

## Problem Description
Water is an essential component for maintaining good health and well-being.
However, one main challenge for sustainable development in Eswatini is bridging the
gap in water supply between rural and urban areas, ensuring reliable access to clean
water and sanitation services to all people. Lack of access to clean water and
sanitation can lead to various challenging conditions in our country, mainly health
issues. In spite of the Rural Water Supply Branch (RWSB), a fully-fledged government
department, that is involved with the design, construction, maintenance, repair and
rehabilitation of rural water supply schemes, rural areas have always faced difficulty
in accessing clean water supplies due to lack of timely maintenance of water
schemes. RWSB does provide technical support to communities in the form of
maintenance technicians to assist communities in repairing broken down
components of the water supply schemes. The section also provides training for
community members to carry out minor maintenance of their respective water
supply schemes. However, the problem lies in accurately capturing information
about the water supply system and making informed decisions in real-time. This
could lead to quick responses in the case the system fails, planned and timely
maintenance ensuring a constant supply of clean water through:
● Remote monitoring
● Remote control

## The LoRa32 sender code:
This is the code that runs on the arduino LoRa32 board that sends data from the instrumentation room (at the water source eg: stream) to the control room where it is uploaded to the realtime database, which harvests the data for the stream through sensors intrefaced to the board. This board has no internet connection only radio communicate with the other board at the control room. The connected sensors are:

1. Turbidity sensor which detect the amoount of suspended particles in water.test for dissolved chemicals in the water.
2. pH sensor that tests the pH of water.
3. TSD  that test for dissolved chemicals in the water.
4. Ultrasonic sensor.

[Find the arduino.ion code](https://github.com/minenhledlamini12/Rural-smart-water/tree/main/firebase-rural-smart-water) That collects and send the water parameters to the other board at the control room.

This sensors inputs the readings to the arduino board which then sends the data to the other board to be transmitted to
google firebase databese which will display on the webapp. The board also controls the the pump, the valves that are in the water supply system.

## The LoRa32 receiver code:
This is the code that runs on the arduino LoRa32 board at the control room that receives data from the instrumentation room (at the water source eg: river). The recieved data is uploaded to firebase by this board which has wifi connection to upload the data received from the other board to the realtime database (on firebase). 

[Find the arduino.ion code](https://github.com/minenhledlamini12/Rural-smart-water/tree/main/firebase-rural-smart-water) That recieves and upload the data to firebase.

## The webapp 
The webapp is the dashboard for monitoring and controlling the clean water supply system.

The webapp is developed in vscode. It has the webapp files which are:
1. The webapp html code
2. The css file for app visuals
3. The java scripts file


This is the html, css, java script code that makes the webapp for visualizing and controlling the system, it was edited with visual studio code and deployed to firebase to fetch and send values to the firebase database and the app, which then communicate with the arduino board that controls the actual components on the ground.



[Find the vscode codes](https://github.com/minenhledlamini12/Rural-smart-water/tree/main/firebase-rural-smart-water) The folder shows all the files running on vscode to make the webapp.

## Google Earth Engine code
This is the google erath code that locate the exact location of a resevoir to see the real set up of the source.  
[Find the code](https://github.com/minenhledlamini12/Rural-smart-water/blob/main/rural_smart_water_googleearth.py)


