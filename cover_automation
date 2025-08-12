let CONFIG = {
  apiKey: "YOUR_OPENWEATHERMAP_API_KEY", // API key for OpenWeatherMap
  lat: 52.2667, // Latitude (e.g., Cremlingen, Germany)
  lon: 10.8667, // Longitude (e.g., Cremlingen, Germany)
  tempThreshold: 26, // Temp threshold for lowering blinds (°C)
  cloudThreshold: 50, // Max cloud cover % for lowering blinds
  cloudConsecutiveThreshold: 3, // Consecutive cloud checks for raising blinds
  eveningHour: 18, // Hour to pause and raise blinds
  startHour: 10, // Hour to resume operation
  closingPosition: 15, // Position for closing blinds (%)
  openingPosition: 70, // Position for opening blinds (%)
  checkInterval: 300000 // Interval for timer in milliseconds (e.g., 10000 = 10 seconds)
};
print("Config loaded:", JSON.stringify(CONFIG));

let consecutiveCloudCount = 0; // Count of consecutive cloud detections
let timer_handle = null; // Timer handle for control

function checkWeather() {
  print("Before HTTP call");
  let hour = new Date().getHours();
  let minute = new Date().getMinutes();
  print("Current hour:", hour, "minute:", minute);
  if (hour < CONFIG.startHour || hour >= CONFIG.eveningHour) {
    print("Outside operating hours, checking next start");
    if (hour >= CONFIG.eveningHour) {
      print("After evening hour, raising to " + CONFIG.openingPosition + "% until tomorrow " + CONFIG.startHour + ":00");
      Shelly.call("Cover.GoToPosition", {id: 0, pos: CONFIG.openingPosition}, function(err) {
        if (err) print("Error raising to " + CONFIG.openingPosition + "%:", err);
        else print("Blinds successfully set to " + CONFIG.openingPosition + "%");
      });
      if (timer_handle) Timer.clear(timer_handle);
      let now = new Date();
      let delay = 0;
      try {
        if (hour >= CONFIG.startHour) {
          delay = (24 - hour + CONFIG.startHour) * 60 * 60 * 1000 - minute * 60 * 1000;
          if (delay < 0) delay += 24 * 60 * 60 * 1000;
        } else {
          delay = (CONFIG.startHour - hour) * 60 * 60 * 1000 - minute * 60 * 1000;
          if (delay < 0) delay += 24 * 60 * 60 * 1000;
        }
        timer_handle = Timer.set(delay, true, checkWeather);
        print("Next run in", delay / 1000 / 60 / 60, "hours (approx. " + CONFIG.startHour + ":00 tomorrow)");
      } catch (e) {
        print("Timer calc error:", e);
        timer_handle = Timer.set(CONFIG.checkInterval, true, checkWeather);  // Fallback with config interval
        print("Fallback timer started at " + (CONFIG.checkInterval / 1000 / 60) + " mins with handle:", timer_handle);
      }
      return;
    }
    return;
  }

  print("Within operating hours, performing HTTP call");
  Shelly.call("HTTP.GET", {
    url: "https://api.openweathermap.org/data/2.5/weather?lat=" + CONFIG.lat + "&lon=" + CONFIG.lon + "&appid=" + CONFIG.apiKey + "&units=metric",
    timeout: 10000
  }, function(res) {
    print("After HTTP call, response code:", res.code);
    if (res.code === 200) {
      try {
        let data = JSON.parse(res.body);
        let temp = data.main.temp;
        let clouds = data.clouds.all;
        let weather = data.weather[0].main;
        print("Current values - Temp:", temp, "°C, Clouds:", clouds, "%, Weather:", weather);

        if (temp > CONFIG.tempThreshold && clouds < CONFIG.cloudThreshold && 
            (weather === "Clear" || weather === "Clouds")) {
          print("Condition for lowering met (sunshine or light clouds detected)");
          Shelly.call("Cover.GoToPosition", {id: 0, pos: CONFIG.closingPosition}, function(err) {
            if (err) print("Error lowering to " + CONFIG.closingPosition + "%:", err);
            else print("Blinds successfully set to " + CONFIG.closingPosition + "%");
          });
          consecutiveCloudCount = 0;  // Reset counter
        } else {
          print("Condition for lowering not met");
          if (temp <= CONFIG.tempThreshold) {
            print("Temp too low, raising to " + CONFIG.openingPosition + "%");
            Shelly.call("Cover.GoToPosition", {id: 0, pos: CONFIG.openingPosition}, function(err) {
              if (err) print("Error raising to " + CONFIG.openingPosition + "%:", err);
              else print("Blinds successfully set to " + CONFIG.openingPosition + "%");
            });
            consecutiveCloudCount = 0;  // Reset counter
          } else if (clouds >= CONFIG.cloudThreshold) {
            consecutiveCloudCount += 1;  // Increment counter
            print("Clouds detected, consecutive count:", consecutiveCloudCount);
            if (consecutiveCloudCount >= CONFIG.cloudConsecutiveThreshold) {
              print("Clouds detected multiple times, raising to " + CONFIG.openingPosition + "%");
              Shelly.call("Cover.GoToPosition", {id: 0, pos: CONFIG.openingPosition}, function(err) {
                if (err) print("Error raising to " + CONFIG.openingPosition + "%:", err);
                else print("Blinds successfully set to " + CONFIG.openingPosition + "%");
              });
              consecutiveCloudCount = 0;  // Reset counter
            }
          } else {
            consecutiveCloudCount = 0;  // Reset counter
            print("No change, counter reset");
          }
        }
      } catch (e) {
        print("JSON error:", e);
      }
    } else {
      print("HTTP error: Code", res.code, "Message:", res.message);
    }
  }, function(err) {
    print("HTTP call error:", err);
  });
}

timer_handle = Timer.set(CONFIG.checkInterval, true, checkWeather);  // Use config interval
print("Timer started with handle:", timer_handle);
