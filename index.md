## Human Assist Multi Purpose Robot

### Features
- Autonomous multi purpose robot 
- Exploration of wifi based tracking and indoor localization

```markdown
Code block 
```
#include "mbed.h"
#include "Motor.h"

Motor right(p23, p6, p5); // pwm, fwd, rev
Motor left(p24, p8, p7); // pwm, fwd, rev
Serial Blue(p13,p14); // BLE control
Serial  pi(USBTX, USBRX); // Communication with RPi 
DigitalOut led1(LED1); // Data receive indicator in auto mode

//global variables for main and interrupt routine
float robo_speed = 0.5;
float right_robo_speed = 0.6;
volatile bool auto_mode = false;
volatile bool button_ready = 0;
volatile int  bnum = 0;
volatile int  bhit  ;
volatile int s1_intensity = 0, s1_intensity_prev = 0;
volatile bool dir_change = false;
volatile int count = 0;
volatile char  temp[4] = {0 , 0, 0, 0};
//state used to remember previous characters read in a button message
enum statetype {start = 0, got_exclm, got_B, got_num, got_hit};
statetype state = start;


//Interrupt routine to parse message with one new character per serial RX interrupt
void parse_message()
{
    switch (state) {
        case start:
            if (Blue.getc()=='!') state = got_exclm;
            else state = start;
            break;
        case got_exclm:
            if (Blue.getc() == 'B') state = got_B;
            else state = start;
            break;
        case got_B:
            bnum = Blue.getc();
            state = got_num;
            break;
        case got_num:
            bhit = Blue.getc();
            state = got_hit;
            break;
        case got_hit:
            if (Blue.getc() == char(~('!' + ' B' + bnum + bhit))) button_ready = 1;
            state = start;
            break;
        default:
            Blue.getc();
            state = start;
    }
}

//
// Move back
//
void m_reverse(void)
{
    left.speed(robo_speed);
    right.speed(robo_speed);
}

//
// Move forward
// Slight change in left and right speeds to compensate
// for the wheel drift on test terrain
//
void m_forward(void)
{
    left.speed((-1.0 * robo_speed) + 0.02);
    right.speed((-1.0 * robo_speed) - 0.02);
}

//
// Move left
//
void m_left(void)
{
    left.speed(robo_speed);
    right.speed(-1.0 * robo_speed);
}

//
// Move right
//
void m_right(void)
{
    right.speed(right_robo_speed);
    left.speed(-1.0 * right_robo_speed);
}

//
// Stop both wheels
//
void stop(void)
{
    right.speed(0.0);
    left.speed(0.0);
}

//
// Serial Interrupt for Raspberry Pi communication
//
void dev_recv()
{
    
  temp[count++] = pi.getc();
  temp[count++] = pi.getc();
  temp[count++] = pi.getc();
  temp[count++] = pi.getc();
  
  if ( count == 4)
  {
       count = 0;
       s1_intensity = ((int)temp[0] - '0') * 10 + ((int)temp[1] - '0');  
  } 
         
  if(auto_mode && (!dir_change))
  {
      led1 = !led1;
            
      if (s1_intensity < 40)
      {
          stop();    
          auto_mode = false;
          state = start;
      }
      else if(s1_intensity > (s1_intensity_prev + 5))
      {
          m_right();   
          wait(0.6);
          m_forward();
          wait(3.0);
          stop();
          s1_intensity_prev = s1_intensity;
      }
      else if (s1_intensity > 65)
      {
          m_right();   
          wait(1.0);
          m_forward();
          wait(4.0);
          stop();                
      }
      else
      {
          m_forward();
          wait(2);
          stop();
      }
          
  }
}


int main()
{
     Blue.attach(&parse_message,Serial::RxIrq);
     pi.baud(9600);
     pi.attach(&dev_recv, Serial::RxIrq);
     
    while(1) {
        if (button_ready) 
        {                          
           if(bnum == '7')
              auto_mode = true;
           else if (bnum == '8')
              auto_mode = false;
           else{}
                    
           if(!auto_mode)
           {
                        switch (bnum) {
                            case '1': //number button 1
                                if (bhit=='1') {
                                    m_forward();
                                } else {
                                    //add release code here
                                    stop();
                                }
                                break;
                            case '2': //number button 2
                                if (bhit=='1') {
                                    m_reverse();
                                } else {
                                    stop();
                                }
                                break;
                            case '3': //number button 3
                                if (bhit=='1') {
                                    m_left();
                                } else {
                                    stop();
                                }
                                break;
                            case '4': //number button 4
                                if (bhit=='1') {
                                    m_right();
                                } else {
                                    stop();
                                }
                                break;
                            case '5': //button 5 up arrow
                                if (bhit=='1') {
                                    robo_speed += 0.05;
                                } else {
                                    
                                }
                                break;
                            case '6': //button 6 down arrow
                                if (bhit=='1') {
                                    robo_speed -= 0.05;
                                } else {
                                    
                                }
                                break;                            
                            default:
                                break;
                        }
                    }       
                }      
                
        
        wait(0.1);
    }
}


### Components used 
