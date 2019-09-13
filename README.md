# ag-brainlab
## Links
- [HC-05 Bluetooth Module](https://www.gme.cz/data/attachments/dsh.772-148.1.pdf)
- [Mindbot Project](https://courses.ece.cornell.edu/ece5990/ECE5725_Fall2018_Projects/sak295_mab586/index.html)
- [Mindwave Library](https://github.com/robintibor/python-mindwave-mobile)
## Description
### Introduction
An EEG, or electroencephalogram is a test that measures the electrical signals output by the brain through the use of electrodes placed on the head of the subject. EEGs were historically reserved for use in the medical field as a diagnostic test. However, in recent years, many engineers and have discovered that there exist many exciting applications for EEG technology in research, control engineering, and consumer products among many others. Thanks to the growth in this field, technology has not only improved, but so have resources for independent developers and price points of commercial headsets. One such headset, the MindFlex, is available at a very low cost as part of a fun family game. However we think that this headset is capable of much more than just a game controller. With “MindBot” we hope to show some of the possibilities that exist for use of EEG interfaces in modern technology and future advancements.

Generic placeholder image
### Project Objective:
Hack a mind flex headset for general EEG use.
Show proof of concept for applications of low cost EEG technology.
Control a mechanical system using two distinct signals.
Design
Our project was broken down into 4 main phases: Hacking the Mindflex headset, Obtaining Data via Bluetooth, and Data Processing (in the form of the robot), and the GUI. In figure 1 you can see the Mindflex headset with the attached bluetooth module. In figure 2 you can see the robot with the attached raspberry pi and TFT touchscreen.

### Mindflex Headset
Fig 1: Headset with attached HC05

### Hacking the Headset
Out of the box, the chip inside the mind flex headset is only set up to communicate with the board inside the base of the game. In order to communicate with the chip, we had to attach our own bluetooth module directly to the board. We got this idea from a tutorial on instructables.com. This involved soldering jumper wires to the TX, RX, V+ and GND of the chip. We then connected the V+ and GND accordingly; the TX of the chip went to the RX of our bluetooth module, and the RX of the chip went to the TX of our bluetooth module. During this process we considered multiple different bluetooth modules, including the HC06 and the Adafruit Bluefruit LE UART Friend. Between the two HC models, the HC05 is much more user friendly. The Friend is actually the most user friendly, as it is programmable via standard FTDI cables and comes with many accessible libraries. However, in order to dynamically switch between receiving and transmitting data to over bluetooth, it requires an external circuit. The Neurosky chip within the headset has multiple modes of data transmission, out of the box, the chip in the mindflex is automatically configured for mode 1, which is data that has already been processed via fast fourier transform and is transmitted at a baud rate of 9600. In order to get raw data, the chip needs to be switched to mode 2 and will transmit at a baud rate of 57600. One way to do this is to remove a surface mount resistor (see the Instructable for more information). Another way to do this is to transmit a hex byte from the pi to the Neurosky chip. We were wary of the first method as we did not want to irreperably damage the headset, but the second method does not survive power cycling. This means that we would have to transmit the byte every time the headset was powered up. For this reason, we felt it would be more prudent to go with a chip capable of dynamically switching modes.

Screen
Fig 2: Raspberry pi attached to robot

### Bluetooth
The trickiest part of implementation of the MindBot was in creating a reliable bluetooth connection that we could read the data from. After a significant amount of trial and error, the best route ended up being to create a socket connection through the rfcomm port using the RPi as a host and the HC05 as a client. There is no standard UUID protocol for the Neurosky chip, so we used existing mindwave libraries to read directly from the buffer, and altered the libraries as needed to fit our application.

### Data Processing
In order to move the bot, we read the wearer's attention level and correlate that to a speed. We wanted to switch between moving straight and turning when the user blinks. When there is a peak the user has blinked. The first part of this involved reading in the data from the neurosky chip. Each data byte is proceeded by a hex row code indicating what type of data is contained within that byte. There are two row codes available for attention level and blink strength. Other row codes include raw data, meditation level, an EEG power spectrum, and a value indicating a power electrical connection between the electrode in the headseat and the wearer. The libraries we were using to process the data created an object with a class based on the row code for each data point. We would read in the byte, check the class, and if it was an attention level data point, we would update the speed and the gui then move the robot. If the data point was a blink, we would update the direction of one of the servos, then move the bot.

Start
Fig 3: Start Screen

According to the servo datasheets, a neutral position occurs when the servo is sent a pulse width of 1.3 ms with a 20 ms gap between each pulse. This corresponds to a 7.5 percent duty cycle. Maximum clockwise rotation is achieved with a pulse width of 1.5 ms, which corresponds to an 8.5 percent duty cycle, while a maximum counterclockwise rotation corresponds to a 6.5 percent duty cycle. The attention level received has a range of 0 to 100. we multiply this percentage by the difference between the maximum speed duty cycle and the neutral duty cycle (1%) and add that value to the neutral signal. When we do this, we make sure that the servo continues rotating in the same direction, but this allows the speed to vary linearly with concentration level.

As we worked, we discovered that the mindflex headset is not configured to perform blink detection, meaning that although the chip is capable of doing so, the chip was configured so that it never output a byte with a row code corresponding to a blink. However the EEG power spectrum byte contains values representing each of 8 different types of waves, delta, theta, low alpha, high alpha, low beta, high beta, low gamma, and mid gamma. Upon researching, we found that a blink causes a spike in the amplitude of brain waves between the 8 types of brain waves. In order to detect a blink, we calculate the average value of the wearer's mind waves over the time they've been wearing the headseat and compare that to the current average of their brain waves. If the difference is above a certain threshhold, we consider that event a blink. Therefore we adjusted our algorithm to enter the blink detection function when we received a data point with a row code indicating an EEG power spectrum byte. The implementation of this code may be seen in the appendix.

One of the tough parts about the blink detection is that the data is received from the headset at a rate of roughly 1 Hz. This means if the user blinks in between data points being received, the data indicating a spike in brain activity will not be transmitted to the raspberry pi. This is a hardware limitation of the mindflex headset and could possibly be combatted using a more expensive model that is capable of continuous data streaming, however that would lie outside the scope of this project as our goal is to show the possiblities of a low cost module.

Screen
Fig 4: Screen displaying concentration level

### GUI
The program starts as soon as the pi is powered up. So in order to make it easier for the user, we wanted them to be able to start the bot when they were ready. The start screen features two buttons that the user may use to either start the bot or to quit the application. In the event that the touch screen is in-operable, the user may use the physical buttons on the side of the pi TFT as well. Button #27 is to quit, while button #17 is to start. Once the user starts the robot, the default is for it to begin moving forward.

At this point we switch the GUI to display the current state of the user. We want the user to know their current attention level in order to help them focus, or unfocus, their thoughts in order to move the bot the speed they wish, so we print the attention percentage to the screen eery time it is updated. This also helps the user know which techniques to focus or distract themselves are working, such as focusing on the motion of the bot, doing math problems. or letting their mind wander. The quit button on screen is still available to exit the application, and the physical quit button, #27, still works in this mode as well.

### Result
We were able to successfully use the users attention level for variable speed control of the robot, with a statistically significant degree of accuracy in using blinking to toggle the degree of freedom of the motion. Moving forward, we want to use the raw data to perform more accurate blink detection, potentially using machine learning to create a blink threshhold that is more accurate to the current user as they wear the headset, as opposed to the current method which employs a global threshhold for all users. We would also like to add in features that display the state of all 6 of the users brain waves. We would also potentially like to build a maze for wearers to move the bot through and perform a study on how effective the module could be for brain training.
