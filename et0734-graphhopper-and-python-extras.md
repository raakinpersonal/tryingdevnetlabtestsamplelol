# ET0734 Extras: Graphhopper (4.9.2) and Python (1.3.3, 3.4.6, 3.1.12)

Notes that are NOT on the 2 page printed cheat sheet. Built from the official lab PDFs.

---

## Part A: Lab 4.9.2 Graphhopper, Integrate a REST API in a Python Application

### What the app does, in plain words

Graphhopper is a directions engine (like the brain behind Google Maps). The app asks for a vehicle, a starting location and a destination, then:

1. **Geocoding**: turns each place name into map coordinates (latitude and longitude), because routing maths needs numbers, not names.
2. **Routing**: asks for the route between the two coordinate points, and displays distance, trip duration and turn by turn instructions.

Folder: `~/labs/devnet-src/graphhopper`. The script is built in versions, saving a new file each time: `graphhopper_parse-json_1.py` up to `_7.py`. Each version adds one feature, so if a later version breaks you can fall back to the previous one.

Version map:
- `_1`: imports, build the geocode URL, GET it, print the raw JSON
- `_2`: the `geocoding()` function, plus name / state / country display
- `_3`: user input with a `while True` loop, and `q` or `quit` to exit
- `_4`: error handling for empty results (see the IndexError trap below)
- `_5`: routing URL, distance and duration, with unit conversions
- `_6` and `_7`: vehicle profile choice and the turn by turn instructions list

### Getting an API key (Part 3)

1. Go to graphhopper.com and Sign Up. For Company, enter: Cisco Networking Academy Student
2. Verify the email (check the spam folder), log in, click API Keys, then Add API Key
3. Copy the key into a text file. The key goes inside every URL the app builds. A wrong key gives status **401**.

### Geocoding: the URL and the JSON

```python
import requests
import urllib.parse

geocode_url = "https://graphhopper.com/api/1/geocode?"
route_url = "https://graphhopper.com/api/1/route?"
key = "your_api_key"
```

`urllib.parse.urlencode()` builds the parameter part of the URL and converts special characters, so a space becomes `+` and a comma becomes `%2C`.

Example built URL:
```
https://graphhopper.com/api/1/geocode?q=Washington%2C+D.C.&limit=1&key=YOUR_KEY
```

The reply JSON has two top level keys, `hits` and `locale`. The coordinates live inside the first hit:

```python
lat = json_data["hits"][0]["point"]["lat"]
lng = json_data["hits"][0]["point"]["lng"]
```

Other useful fields in `hits[0]`: `name`, `country`, `state`, `osm_value` (the location type, for example `city`). Note: `state` and `country` are NOT present for every location, so the lab checks `"country" in json_data["hits"][0]` before using them.

### The geocoding() function (version _2)

```python
def geocoding(location, key):
    geocode_url = "https://graphhopper.com/api/1/geocode?"
    url = geocode_url + urllib.parse.urlencode({"q": location, "limit": "1", "key": key})

    replydata = requests.get(url)
    json_data = replydata.json()
    json_status = replydata.status_code

    if json_status == 200 and len(json_data["hits"]) != 0:
        lat = json_data["hits"][0]["point"]["lat"]
        lng = json_data["hits"][0]["point"]["lng"]
    else:
        lat = "null"
        lng = "null"
    return json_status, lat, lng
```

Called like this. It returns a tuple, so `orig[0]` is the status, `orig[1]` is lat, `orig[2]` is lng:

```python
orig = geocoding(loc1, key)
dest = geocoding(loc2, key)
```

### The big trap: empty hits

A nonsense location (for example `slkdjf`) still returns status **200**, but with an empty list: `hits: [ ]`. If the code then tries `json_data["hits"][0]`, Python crashes with **IndexError: list index out of range**. That is why the test condition is:

```python
if json_status == 200 and len(json_data["hits"]) != 0:
```

Both halves must be true: the request succeeded AND at least one result came back.

### User input and quitting (version _3)

```python
while True:
    loc1 = input("Starting Location: ")
    if loc1 == "quit" or loc1 == "q":
        break
    orig = geocoding(loc1, key)

    loc2 = input("Destination: ")
    if loc2 == "quit" or loc2 == "q":
        break
    dest = geocoding(loc2, key)
```

Without the quit check, the only way out is Ctrl+C, which gives a KeyboardInterrupt traceback.

### Routing: building the URL (version _5)

The route URL needs the two points as `lat,lng` pairs. The comma between them is written `%2C`:

```python
if orig[0] == 200 and dest[0] == 200:
    op = "&point=" + str(orig[1]) + "%2C" + str(orig[2])
    dp = "&point=" + str(dest[1]) + "%2C" + str(dest[2])
    paths_url = route_url + urllib.parse.urlencode({"key": key, "vehicle": vehicle}) + op + dp
    paths_status = requests.get(paths_url).status_code
    paths_data = requests.get(paths_url).json()
```

Example built URL:
```
https://graphhopper.com/api/1/route?key=YOUR_KEY&vehicle=car&point=38.8950368%2C-77.0365427&point=39.2908816%2C-76.610759
```

Vehicle profiles: **car, bike, foot**. If the user types anything else, the final version of the lab resets it to the default, car.

### Reading the route reply: distance, time, instructions

The route JSON has three top level keys: `hints`, `info`, `paths`. Everything useful is in `paths[0]`:

- `paths_data["paths"][0]["distance"]` is in **metres**
- `paths_data["paths"][0]["time"]` is in **milliseconds**

Conversions used in the lab:

```python
km = paths_data["paths"][0]["distance"] / 1000
miles = paths_data["paths"][0]["distance"] / 1000 / 1.61
print("Distance Traveled: {0:.1f} miles / {1:.1f} km".format(miles, km))

sec = int(paths_data["paths"][0]["time"] / 1000 % 60)
min = int(paths_data["paths"][0]["time"] / 1000 / 60 % 60)
hr = int(paths_data["paths"][0]["time"] / 1000 / 60 / 60)
print("Trip Duration: {0:02d}:{1:02d}:{2:02d}".format(hr, min, sec))
```

`{:.1f}` means 1 decimal place. `{:02d}` means a 2 digit whole number with a leading zero, so 5 minutes prints as `05`. The `%` (modulo) operator returns the remainder, which is how seconds and minutes are kept under 60.

Turn by turn instructions are a list inside the path. Each instruction has a `text` and a `distance` in metres:

```python
for i in range(len(paths_data["paths"][0]["instructions"])):
    instruction = paths_data["paths"][0]["instructions"][i]["text"]
    distance = paths_data["paths"][0]["instructions"][i]["distance"]
    print("{0} ( {1:.1f} km / {2:.1f} miles )".format(instruction, distance / 1000, distance / 1000 / 1.61))
```

Example output lines:
```
Continue onto 15th Street Northwest ( 0.4 km / 0.3 miles )
Keep left onto Baltimore-Washington Parkway and drive toward Baltimore ( 51.6 km / 32.1 miles )
Arrive at destination ( 0.0 km / 0.0 miles )
```

### Status codes to remember for 4.9.2

- **200**: request worked (but still check hits is not empty for geocoding)
- **401**: bad or missing API key
- Empty `hits`: the place name was not found, usually a typo

---

## Part B: Python extras

### Lab 1.3.3: data types and variables

Check any type with `type()`:

```python
type(98)        # int, a whole number
type(98.6)      # float, has a decimal point
type('Hi!')     # str, text in quotes
type(True)      # bool, True or False
```

Maths operators: `+` add, `-` subtract, `*` multiply, `/` divide (always gives a float, so `20/5` is `4.0`), `**` power (so `3**2` is 9). Order of operations is PEMDAS: Parentheses, Exponents, Multiplication and Division, Addition and Subtraction.

String plus int rule: you cannot join text and a number with `+`.

```python
x = 3
print('The value of x is ' + x)        # TypeError: can only concatenate str (not "int") to str
print('The value of x is ' + str(x))   # works: The value of x is 3
```

Lists and dictionaries:

```python
my_list = ["router", "switch", "server"]   # ordered, index starts at 0
my_list[0]          # "router"
my_list.append("firewall")
len(my_list)        # number of items

my_dict = {"hostname": "R1", "ip": "10.0.0.1"}   # key: value pairs
my_dict["ip"]       # "10.0.0.1", look up by key, not by index
```

`input()` always returns a **string**, even if the user types a number. Convert it before maths:

```python
age = input("Enter your age: ")
age = int(age)
```

Control flow reminders: `if / elif / else` decide, `for` loops over a list or a `range()`, `while` loops until its condition becomes false. Indentation is part of the syntax in Python, 4 spaces, and a wrong indent is an error, not just untidy.

File access:

```python
file = open("data.txt", "r")   # "r" read, "w" write, "a" append
print(file.read())
file.close()
```

### Lab 3.4.6: functions, methods and classes

- A **function** is a standalone block of code, called by name.
- A **method** lives inside a class and can only be called through an object of that class.
- A **class** is a blueprint. Creating an object from it is called instantiating.
- `__init__` runs automatically when an object is created. `self` refers to that specific object, and `self.something` stores data on it.

Function example (myCity.py):

```python
def myCity(city):
    print("I live in " + city + ".")

myCity("Austin")   # I live in Austin.
```

Class example (myLocation.py, the Location class):

```python
class Location:
    def __init__(self, name, country):
        self.name = name
        self.country = country

    def myLocation(self):
        print("Hi, I am in " + self.name + ", " + self.country + ".")

loc = Location("Bali", "Indonesia")
loc.myLocation()   # Hi, I am in Bali, Indonesia.
```

Circle class reminder: a method can **return** a value instead of printing it. `circumference` returns `3.14 * 2 * self.radius`, so the caller wraps it in `print(...)`. Radius 2, 5 and 7 give 12.56, 31.4 and 43.96 (radius 5 may display as 31.400000000000002, that is normal float display, not an error).

### Lab 3.1.12: development tools extras

The venv and pip basics are on the printed sheet. The extra idea in this lab is **sharing** an environment: `pip3 freeze > requirements.txt` records every installed package with its exact version, and anyone can rebuild the same environment from that file:

```bash
python3 -m venv newenv
source newenv/bin/activate
pip3 install -r requirements.txt
```

The freeze output includes helper packages you never typed (installing `requests` also brought in certifi, charset-normalizer, idna and urllib3). That is normal: pip installs dependencies automatically, and freeze records everything actually in the environment.

Unit testing idea from this lab: a test file imports `unittest`, defines test methods that use `assertEqual(expected, actual)`, and runs with `python3 -m unittest`. Tests pass silently apart from an OK summary, and a failure shows exactly which values did not match.
