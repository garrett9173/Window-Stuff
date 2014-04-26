#include <genieArduino.h>
#include <stdio.h>
#include <stdint.h>
#include <ctype.h>

int slider0;
int thermometer0;
int strings0;
int sound;
int tempOut;
int leddigits0;
const int shadesPin = 52;
int shades = LOW;
int slider1;


Genie genie;
#define RESETLINE 4

void setup ()
{
  Serial.begin(200000);
  genie.Begin(Serial);
  
  genie.AttachEventHandler(myGenieEventHandler);
  
  // Reset the Display (change D4 to D2 if you have original 4D Arduino Adaptor)
  // If NOT using a 4D Arduino Adaptor, digitalWrites must be reversed as Display Reset is Active Low, and
  // the 4D Arduino Adaptors invert this signal so must be Active High.
  pinMode(RESETLINE, OUTPUT);  // Set D4 on Arduino to Output (4D Arduino Adaptor V2 - Display Reset)
  digitalWrite(RESETLINE, 1);  // Reset the Display via D4
  delay(100);
  digitalWrite(RESETLINE, 0);  // unReset the Display via D4

  delay (3500); //let the display start up after the reset (This is important)

  //Turn the Display on (Contrast) - (Not needed but illustrates how)
  genie.WriteContrast(1); // 1 = Display ON, 0 = Display OFF
  
  pinMode(shadesPin, OUTPUT);
}


void loop()
{
  static long waitPeriod = millis();
  static int gaugeAddVal = 1;
  static int gaugeVal = 69;
  
  genie.DoEvents();

  if (millis() >= waitPeriod)
  {
    // Write to CoolGauge0 with the value in the gaugeVal variable
    genie.WriteObject(GENIE_OBJ_COOL_GAUGE, 0, gaugeVal);

    // The results of this call will be available to myGenieEventHandler() after the display has responded
    // Do a manual read from the UserLEd0 object
    genie.ReadObject(GENIE_OBJ_USER_LED, 0x00);
    //genie.WriteStr(0, quack);
    
    waitPeriod = millis() + 50; // rerun this code to update Cool Gauge and Slider in another 50ms time.
  }
}

/////////////////////////////////////////////////////////////////////
//
// This is the user's event handler. It is called by genieDoEvents()
// when the following conditions are true
//
//		The link is in an IDLE state, and
//		There is an event to handle
//
// The event can be either a REPORT_EVENT frame sent asynchrounously
// from the display or a REPORT_OBJ frame sent by the display in
// response to a READ_OBJ request.
//

/*COMPACT VERSION HERE, LONGHAND VERSION BELOW WHICH MAY MAKE MORE SENSE
void myGenieEventHandler(void)
{
  genieFrame Event;
  int slider_val = 0;
  const int index = 0;  //HARD CODED TO READ FROM Index = 0, ie Slider0 as an example

  genieDequeueEvent(&Event);

  //Read from Slider0 for both a Reported Message from Display, and a Manual Read Object from loop code above
  if (genieEventIs(&Event, GENIE_REPORT_EVENT, GENIE_OBJ_SLIDER, index) ||
    genieEventIs(&Event, GENIE_REPORT_OBJ,   GENIE_OBJ_SLIDER, index))
  {
    slider_val = genieGetEventData(&Event);  // Receive the event data from the Slider0
    genieWriteObject(GENIE_OBJ_LED_DIGITS, 0x00, slider_val);     // Write Slider0 value to to LED Digits 0
  }
}*/

//LONG HAND VERSION, MAYBE MORE VISIBLE AND MORE LIKE VERSION 1 OF THE LIBRARY
void myGenieEventHandler(void)
{
  genieFrame Event;
  genie.DequeueEvent(&Event);

  int shadesMotor = 0;
  bool UserLed0;
  

  //If the cmd received is from a Reported Event (Events triggered from the Events tab of Workshop4 objects)
  if (Event.reportObject.cmd == GENIE_REPORT_EVENT)
  {
    if (Event.reportObject.object == GENIE_OBJ_SLIDER)                // If the Reported Message was from a Slider
    {
      if (Event.reportObject.index == 1)                              // If Slider1
      {
        shadesMotor = genie.GetEventData(&Event);                      // Receive the event data from the Slider0

        if(shadesMotor == 1){
          shades = HIGH; 
        }else(shades = LOW);        
        
        digitalWrite(shadesPin, shades);
        
      }
    }
  }

  //Open close window using slider
  if (Event.reportObject.cmd == GENIE_REPORT_EVENT)
  {
    if (Event.reportObject.object == GENIE_OBJ_SLIDER)                // If the Reported Message was from a Slider
    {
      if (Event.reportObject.index == 0)                              // If Slider0
      {
        int slider_val = genie.GetEventData(&Event);                      // Receive the event data from the Slider0
              
        String stringOne = "Set temp to: ";
        stringOne += slider_val + 50;
        stringOne += " deg F";
        char setTemp[50];
        stringOne.toCharArray(setTemp, 50);
        genie.WriteStr(0, setTemp);
        
        if(slider_val == 40){
          shades = HIGH;
          tone(3,440);
        }else(shades = LOW);
        
        digitalWrite(shadesPin, shades);
      }
    }
  }

  //Manual override open/close window
  if (Event.reportObject.cmd == GENIE_REPORT_OBJ)
  {
    if (Event.reportObject.object == GENIE_OBJ_4DBUTTON)              // If the Reported Message was from a 4Dbutton
    {
      if (Event.reportObject.index == 0)                              // If 4Dbutton0
      {
        bool overRide = genie.GetEventData(&Event);
        if (overRide == 0) 
        {
          UserLed0 = !UserLed0;
          genie.WriteStr(0, "Window override");
          genie.WriteObject(GENIE_OBJ_USER_LED, 0x00, UserLed0);
          
        }
      }
    }
  }



  //If the cmd received is from a Reported Object, which occurs if a Read Object (genie.ReadOject) is requested in the main code, reply processed here.
  /*if (Event.reportObject.cmd == GENIE_REPORT_OBJ)
  {
    if (Event.reportObject.object == GENIE_OBJ_USER_LED)              // If the Reported Message was from a User LED
    {
      if (Event.reportObject.index == 0)                              // If UserLed0
      {
        bool UserLed0_val = genie.GetEventData(&Event);               // Receive the event data from the UserLed0
        UserLed0_val = !UserLed0_val;                                 // Toggle the state of the User LED Variable
        genie.WriteObject(GENIE_OBJ_USER_LED, 0x00, UserLed0_val);    // Write UserLed0_val value back to to UserLed0
      }
    }
  }*/

  //This can be expanded as more objects are added that need to be captured

  //Event.reportObject.cmd is used to determine the command of that event, such as an reported event
  //Event.reportObject.object is used to determine the object type, such as a Slider
  //Event.reportObject.index is used to determine the index of the object, such as Slider0
  //genie.GetEventData(&Event) us used to save the data from the Event, into a variable.
}
