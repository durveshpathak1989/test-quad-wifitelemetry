# TelemetryWiFi changes for HighRateLogger endpoints

These changes assume your `TelemetryWiFi` library uses the standard ESP32 `WebServer` class. If your version uses `ESPAsyncWebServer`, use the alternate section at the end.

## A. Changes in `TelemetryWiFi.h`

Add callback typedefs near your other provider typedefs:

```cpp
typedef String (*TextProviderFn)();
typedef void   (*VoidHandlerFn)();
```

If you already have equivalent typedefs, reuse them.

Inside the `TelemetryWiFi` class public section, add:

```cpp
void setFlightLogCsvProvider(TextProviderFn fn);
void setFlightLogStatusProvider(TextProviderFn fn);
void setFlightLogResetHandler(VoidHandlerFn fn);
```

Inside the private section, add:

```cpp
TextProviderFn _flightLogCsvProvider = nullptr;
TextProviderFn _flightLogStatusProvider = nullptr;
VoidHandlerFn  _flightLogResetHandler = nullptr;
```

## B. Changes in `TelemetryWiFi.cpp`

Add these setter implementations:

```cpp
void TelemetryWiFi::setFlightLogCsvProvider(TextProviderFn fn)
{
    _flightLogCsvProvider = fn;
}

void TelemetryWiFi::setFlightLogStatusProvider(TextProviderFn fn)
{
    _flightLogStatusProvider = fn;
}

void TelemetryWiFi::setFlightLogResetHandler(VoidHandlerFn fn)
{
    _flightLogResetHandler = fn;
}
```

Inside `TelemetryWiFi::begin(...)`, where your other routes are registered, add:

```cpp
_server.on("/flightlog.csv", HTTP_GET, [this]() {
    if (!_flightLogCsvProvider) {
        _server.send(503, "text/plain", "flight log CSV provider not registered");
        return;
    }

    String csv = _flightLogCsvProvider();
    _server.sendHeader("Content-Disposition", "attachment; filename=flightlog.csv");
    _server.send(200, "text/csv", csv);
});

_server.on("/flightlog/status", HTTP_GET, [this]() {
    if (!_flightLogStatusProvider) {
        _server.send(503, "application/json", "{\"error\":\"flight log status provider not registered\"}");
        return;
    }

    _server.send(200, "application/json", _flightLogStatusProvider());
});

_server.on("/flightlog/reset", HTTP_POST, [this]() {
    if (!_flightLogResetHandler) {
        _server.send(503, "application/json", "{\"error\":\"flight log reset handler not registered\"}");
        return;
    }

    _flightLogResetHandler();
    _server.send(200, "application/json", "{\"ok\":true,\"message\":\"flight log reset\"}");
});

// Optional convenience for browsers, because typing a POST is annoying.
_server.on("/flightlog/reset", HTTP_GET, [this]() {
    if (!_flightLogResetHandler) {
        _server.send(503, "application/json", "{\"error\":\"flight log reset handler not registered\"}");
        return;
    }

    _flightLogResetHandler();
    _server.send(200, "application/json", "{\"ok\":true,\"message\":\"flight log reset\"}");
});
```

Your `TelemetryWiFi::update()` should already call:

```cpp
_server.handleClient();
```

Keep that as-is.

## C. If your server member is named differently

Replace `_server` with your actual member name, for example:

```cpp
server.on(...)
server.send(...)
```

## D. ESPAsyncWebServer alternative

If you use `ESPAsyncWebServer`, add routes like this instead:

```cpp
_server.on("/flightlog.csv", HTTP_GET, [this](AsyncWebServerRequest *request) {
    if (!_flightLogCsvProvider) {
        request->send(503, "text/plain", "flight log CSV provider not registered");
        return;
    }

    AsyncWebServerResponse *response = request->beginResponse(200, "text/csv", _flightLogCsvProvider());
    response->addHeader("Content-Disposition", "attachment; filename=flightlog.csv");
    request->send(response);
});

_server.on("/flightlog/status", HTTP_GET, [this](AsyncWebServerRequest *request) {
    if (!_flightLogStatusProvider) {
        request->send(503, "application/json", "{\"error\":\"flight log status provider not registered\"}");
        return;
    }
    request->send(200, "application/json", _flightLogStatusProvider());
});

_server.on("/flightlog/reset", HTTP_POST, [this](AsyncWebServerRequest *request) {
    if (!_flightLogResetHandler) {
        request->send(503, "application/json", "{\"error\":\"flight log reset handler not registered\"}");
        return;
    }
    _flightLogResetHandler();
    request->send(200, "application/json", "{\"ok\":true,\"message\":\"flight log reset\"}");
});

_server.on("/flightlog/reset", HTTP_GET, [this](AsyncWebServerRequest *request) {
    if (!_flightLogResetHandler) {
        request->send(503, "application/json", "{\"error\":\"flight log reset handler not registered\"}");
        return;
    }
    _flightLogResetHandler();
    request->send(200, "application/json", "{\"ok\":true,\"message\":\"flight log reset\"}");
});
```
