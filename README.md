# Node-red to global weather observations website WOW
Node red flow to send MQTT data to the official WOW weather site. located at https://wow.metoffice.gov.uk/

## Background
WeatherObservatiionsWebsite is a site where people all over the world can upload their weather observations and data. In December 2018 the swedish meteological Swedish Meteorological and Hydrological Institute (SMHI) started showing private stations from WOW and using that data to create more accurate predictions. yr.no in norway have been using netatmo sensors for a while in their predictions so it's widely used.

Being a tinkerer and DIY'er I don't have a commercial weather station but WOW supports some that can be bought and there is some software that you can use for others:
* WeatherDisplay - http://www.weather-display.com
* Cumulus - http://sandaysoft.com/products/cumulus
* pywws - https://github.com/jim-easterbrook/pywws
* Weather Snoop - http://www.weathersnoop.com/
* MeteoWare Plus for Netatmo connections - http://plus.meteoware.com/
* Netatmo2WOW - https://github.com/ekkelenkamp/netatmo2wow/
* WeeWX - http://www.weewx.com/

I decided to go with the API method instead since I have a lot of sensors recording the weather and sending the data through MQTT

## Sending the data
### Upload URL
WOW expects an HTTP request, in the form of either GET or POST, to the following URL. When received, WOW will interpret and validate the information supplied and respond as below.
The URL to send your request to is: http://wow.metoffice.gov.uk/automaticreading? followed by a set of key/value pairs indicating pieces of data.
### Mandatory information
All uploads must contain 4 pieces of mandatory information plus at least 1 piece of weather data.

* **Site ID - siteid:**  
The unique numeric id of the site
* **Authentication Key - siteAuthenticationKey:**  
A pin number, chosen by the user to authenticate with WOW.
* **Date - dateutc:**  
Each observation must have a date, in the date encoding specified below.
* **Software Type - softwaretype**  
The name of the software, to identify which piece of software and which version is uploading data

### Date Encoding
The date must be in the following format: YYYY-mm-DD HH:mm:ss, where ':' is encoded as %3A, and the space is encoded as either '+' or %20. An example, valid date would be: 2011-02-29+10%3A32%3A55, for the 2nd of Feb, 2011 at 10:32:55. Note that the time is in 24 hour format. Also note that the date must be adjusted to UTC time - equivalent to the GMT time zone.. This is especially important for users outside the UK, and British users must also take note of the difference between British Summer Time and UTC

### Weather Data:
The following is a list of items of weather data that can be uploaded to WOW. Provide each piece of information as a key/value pair, e.g. winddir=225.5 or tempf=32.2. Note that values should not be quoted or escaped.

Unit	| Key	Value	| Unit
------------- |:-------------:| -----:
baromin	| Barometric Pressure (see note)	| Inch of Mercury
dailyrainin	| Accumulated rainfall so far today	| Inches
dewptf	| Outdoor Dewpoint	| Fahrenheit
humidity	| Outdoor Humidity	| 0-100 %
rainin	| Accumulated rainfall since the previous observation	| Inches
soilmoisture	| % Moisture	| 0-100 %
soiltempf	| Soil Temperature (10cm)	| Fahrenheit
tempf	| Outdoor Temperature	| Fahrenheit
visibility	| Visibility	| Kilometres
winddir	| Instantaneous Wind Direction	| Degrees (0-360)
windspeedmph	| Instantaneous Wind Speed	| Miles per Hour
windgustdir	| Current Wind Gust Direction (using software specific time period)	| 0-360 degrees
windgustmph	| Current Wind Gust (using software specific time period)	| Miles per Hour

### Frequency of submitting observations
It is recommended that the interval between automatic observations from your AWS is at least 5 minutes.

### Pressure & MSLP:
Some weather stations record Mean Sea Level Pressure, and some record Station Pressure (also referred to as Air Pressure). The Site Definition in WOW will be used to determine which pressure measurement to use. If your site specifies a unit for Air Pressure, and MSLP is marked as 'Not Captured', then baromin will be treated as an Air Pressure reading. Otherwise it will be treated as an MSLP reading. To edit your site definition and ensure the correct pressure is selected, visit the edit pages of your site and the options for pressure will be near the bottom. Ensure one is set to 'hectopascals (hPa)' and the other is set to 'not captured'. WOW will then automatically work out the other measurement and store it for you.

### Example URL:
http://wow.metoffice.gov.uk/automaticreading?siteid=123456&siteAuthenticationKey=654321&dateutc=2011-02-02+10%3A32%3A55&winddir=230&windspeedmph=12&windgustmph=12&windgustdir=25&humidity=90&dewptf=68.2&tempf=70&rainin=0&dailyrainin=5&baromin=29.1&soiltempf=25&soilmoisture=25&visibility=25&softwaretype=weathersoftware1.0

## The Node-Red flow
_disclamer: I'm no Node-Red guru so I'm sure there are other and more streamlined versions to do this... feel free to make a pull request with any changes you do!_
![Alt text](/images/wow_weather_allnodes.png?raw=true "Node-red-flow")

### Input nodes
* **Timestamp:**
Adds UTC timestamp every 5 minutes and also sets the interval of sending data
The following nodes in this _Stream_ is to set the timestamp to the encoding mentioned [above](#Date Encoding)
* **Climate:Temp, Climate:HUM, Climate:Pressure**  
These are my mqtt input nodes that I currently have set up. Feel free to add more and change the Join and combine functions.
* **First layer functions**  
These are the ones named **to F** and **to inhg**, since the WOW website requried information in Farenheit and mm merqury we need to change the format from Celcius and hPa
* **Middle stream of nodes**  
yeah, didn't know what I should call it... but it's the flow combining Temperature and Humidity to calculate dew point.
* **Set message topic**  
This is to make it more readable and easier to combine the sensors at a later stage
* **Join**  
This one is important, it combines the different streams and waits until there is something from all of them before it combines them and sends them on. Without this it would send the data but uncomplete, missing bits and parts depending on timing.
* **Combine sensors**  
This combines the different sensors in a function that looks like this  
  ```
  msg.siteid = "replace with your ID";
  msg.auth = "replace with your set password";
  msg.url = "http://wow.metoffice.gov.uk/automaticreading?";
  msg.payload = "siteid=" + msg.siteid + "&siteAuthenticationKey=" + msg.auth + "&dateutc=" + msg.payload.dateutc + "&humidity=" + msg.payload.humidity + "&baromin=" + msg.payload.pressure + "&tempf=" + msg.payload.temp + "&dewptf=" + msg.payload.dewpoint +"&softwaretype=weathersoftware1.0";
  msg.url = msg.url+msg.payload;
  return msg;
  ```
  Replace **msg.siteid** and **msg.auth** with your own information from the WOW website.
  in **msg.payload** it combines the different sensors with the [Weather Data](weather data) that you have and you need to change this to what you are using.
* **Round**  
This takes all the variables and rounds them down to two decimals
   ```
   // List all the variables that you want rounded
   var pressure = msg.payload.pressure;
   var temp = msg.payload.temp;
   var humidity = msg.payload.humidity;
   var dewpoint = msg.payload.dewpoint;
   var dateutc = msg.payload.dateutc;
   // Create a new payload with rounded numbers
   msg.payload = {
     pressure: pressure.toFixed(2),
     temp : temp.toFixed(2),
     humidity: humidity.toFixed(2),
     dewpoint: dewpoint.toFixed(2),
     dateutc: dateutc,
   };
   return msg;
   ```
   The first part creates new variables from you joined values so you have to match these. Once all variables are created it makes a new payload with the rounded numbers. Don't forget that you still need to add the UTC date but with no change here. Otherwise the URL posting will be wrong.

## Links:
Encoding, formatting and data instructions
https://wow.metoffice.gov.uk/support/dataformats

SMHI - Mina observationer - Swedish information
https://www.smhi.se/vadret/vadret-i-sverige/mina-observationer-wow

My site
https://wow.metoffice.gov.uk/observations/details?site_id=24855254-ce02-e911-9462-0003ff598847

## Changelog
**1.0** _(2018-12-19)_
First release
**1.1** _(2018-12-20)_
Added function to round numbers and change the humidity to a number to.
