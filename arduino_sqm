#include <Adafruit_BME280.h>
#include <Adafruit_SSD1306.h>
#include <FreqCounter.h>
#include <Fonts/FreeSerifBold9pt7b.h>

Adafruit_SSD1306 display(128, 64, &Wire);

#define I2C_ADDRESS     0x76
#define PhTCutoff1      2               // boundary between black and dark, used to determine Gate time
#define PhTCutoff2      10              // boundary between dark and light
#define blackperiod     2000            // 2s Gate time, at very dark sites
#define darkperiod      1000            // 1s Gate time, at dark sites
#define lightperiod     100             // 100ms Gate time for day light
#define SAMPLES          5
#define sqm_limit       21.95      
#define PhotoTransPin   A0    
#define BtnPin          4 

Adafruit_BME280 bme; // I2C

float humidity = 0.0;
float pressure = 0.0;
float temperature = 0.0;
float sqm = 0.0;
float dewpoint = 0.0;

double frequency;                         // measured TSL237 frequency which is dependent on light
double irradiance;

int bgLightLevel = 0;
double totalfrequency = 0.0;
double totalirradiance = 0.0;
short period;                             // gate time - specifies duration time for frequency measurement

void setup() {

  Serial.begin(9600);

  pinMode(BtnPin, INPUT_PULLUP);

  display.begin(SSD1306_SWITCHCAPVCC, 0x3C); 
  display.dim(true);
  display.clearDisplay();
  display.display();
  
  // init bme280
  if (!bme.begin(I2C_ADDRESS)) {
    Serial.println(F("Could not find a valid bme280 sensor, check wiring or "
                      "try a different address!"));
    while (1) delay(10);
  }

  /* Default settings from datasheet. */
  bme.setSampling(Adafruit_BME280::MODE_NORMAL,     /* Operating Mode. */
                  Adafruit_BME280::SAMPLING_X2,     /* Temp. oversampling */
                  Adafruit_BME280::SAMPLING_X16,    /* Pressure oversampling */
                  Adafruit_BME280::SAMPLING_X16,    /* Hum. oversampling */
                  Adafruit_BME280::FILTER_X16,      /* Filtering. */
                  Adafruit_BME280::STANDBY_MS_500); /* Standby time. */

  delay(1000);
  
  readBMEValues();
  readSQM();
  draw();
}

void loop() {
  if(digitalRead(BtnPin) == HIGH) {
      
      readBMEValues();
      readSQM();
      draw();
  }
}

void readBMEValues() 
{
  pressure = bme.readPressure()/100;
  temperature = bme.readTemperature();
  humidity = bme.readHumidity();
  calc_dewpoint(temperature, humidity);
}

void drawLoader(int s)
{
  int x = 0;
  int y = 25;
  int width = 124;
  int height = 16;
  
  display.clearDisplay();
  display.drawRect(x, y, width, height, SSD1306_WHITE);
  display.fillRect(x+2, y+2, s*24, height-4, SSD1306_WHITE);
  display.setFont();  
  display.setTextSize(1);  
  if(s < 3) {
      display.setTextColor(SSD1306_WHITE);
  }else {
      display.setTextColor(SSD1306_BLACK);
  }
 
  display.setCursor(60,30);  
  
  String stepStr;
  stepStr.concat(s);
  stepStr.concat(F("/"));
  stepStr.concat(SAMPLES);
  display.println(stepStr);
  
  display.display();
}

void calc_dewpoint(float t, float h)
{
  float logEx;
  logEx = 0.66077 + 7.5 * t / (237.3 + t) + (log10(h) - 2);
  dewpoint = (logEx - 0.66077) * 237.3 / (0.66077 + 7.5 - logEx);
}

void draw(void)
{
  int x = 0;
  int y = 8;
  int p = (int)pressure;
  String sqmStr, temperatureStr, humidityStr, pressureStr, dewpointStr;
  
  sqmStr.concat(F("SQM: "));
  sqmStr.concat(sqm);

  temperatureStr.concat(F("Temperature: "));
  temperatureStr.concat(temperature);
  temperatureStr.concat(F(" C"));

  humidityStr.concat(F("Humidity: "));
  humidityStr.concat(humidity);
  humidityStr.concat(F(" %"));

  pressureStr.concat(F("Pressure: "));
  pressureStr.concat((int)pressure);
  pressureStr.concat(F(" hPa"));

  dewpointStr.concat(F("Dew point: "));
  dewpointStr.concat(dewpoint);
  dewpointStr.concat(F(" C"));
  
  Serial.println(temperatureStr);
  Serial.println(humidityStr);
  Serial.println(pressureStr);
  Serial.println(dewpointStr);
  
  display.clearDisplay();

  display.setCursor(x,y);  
  display.setTextSize(1);  
  display.setTextColor(SSD1306_WHITE);   

  display.setFont(&FreeSerifBold9pt7b);   
  display.print(sqmStr);
  y+=14;
  display.setFont();  
  display.setCursor(x,y);  
  display.print(temperatureStr);
  y+=10;
  display.setCursor(x,y);  
  display.print(humidityStr);
  y+=10;
  display.setCursor(x,y);  
  display.print(pressureStr);
  y+=10;
  display.setCursor(x,y);  
  display.print(dewpointStr);
  display.display();

}

void readSQM() 
{
    int count = 1;
    // refresh sqm readings
    bgLightLevel = analogRead(PhotoTransPin);       
    if ( bgLightLevel < PhTCutoff1 )          
    {
      period = blackperiod;                   // its very dark
    }
    else if ( bgLightLevel < PhTCutoff2 )
    {
      period = darkperiod;                    // its dark
    }
    else
    {
      period = lightperiod;                   // its light
    }

    while(count <= SAMPLES) 
    {
      drawLoader(count);
      readFreq();                               
      irradiance = frequency / 2.3e3;           
      
      // use average reading
      totalfrequency += frequency;
      totalirradiance += irradiance;
      count++;
    }
   
   
    frequency = totalfrequency / SAMPLES;
    irradiance = totalirradiance / SAMPLES;
    totalfrequency = 0.0;
    totalirradiance = 0.0;
    count = 1;

    sqm = (sqm_limit - (2.5 * log10( frequency )) * 0.973); // frequency to magnitudes/arcSecond2
}

void readFreq()
{
  long  pulses = 1L;
  FreqCounter::f_comp = 0;      // Cal Value / Calibrate with professional Freq Counter
  FreqCounter::start(period);   // Gate Time
  while (FreqCounter::f_ready == 0)
  {
    pulses = FreqCounter::f_freq;
  }
  delay(20);
  frequency = ((double)pulses * (double)1000.0) / (double) period;
  // if cannot measure the number of pulses set to freq 1hz which is mag 21.95
  if ( frequency < 1.0 )
  {
    frequency = 1.0;
  }

}
