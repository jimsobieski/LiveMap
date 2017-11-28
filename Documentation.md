# Introduction

Livemap provides a world map with live traffic that users can directly manipulate
to pan and zoom. To make this interaction as fast and responsive as possible, we
chose to pre-render the map at many different levels of detail, and to cut each
map into toxels for quick retrieval and display. This document describes the
projection, coordinate systems, and addressing scheme of the map toxels. 

The word toxel is inspired from voxel, but with time.

# API Essentials

## Map Projection 


In order to make the world map seamless and continuous, and to ensure that map graphics from 
different sources line up properly, **Livemap** uses a single projection 
called **Spherical Mercator projection** for the entire world. It is a widely adopted projection standard for mapping applications (and is used by e.g. OpenStreetMap, Google Maps, Bing Maps and many others). It is a spherical variant of the classical (cylindrical) Mercator projection dating back to 1569. The spherical version has the advantage of being simpler and more computationally effective. The small drawback is that it introduces a small distortion of 0.33% in the north-south direction, but it is so small that it is not visually perceptible. Since the projection is used only for map display, and 
not for displaying numeric coordinates, this deviation is acceptable.

For simplicity, we disregard this small difference and look at the major advantages offered 
by the Mercator projection.

1. The world is a square. Most calculations become simple and can be solved without complex formulas. 
For instance, the distance between two points can be calculated simply using only the Pythagorean theorem.
2. North and south are straight up and down everywhere on the map. Similarly, east and west are always straight left and right everywhere.
3. The shape of relatively small objects are preserved very well, with only an imperceptible deviation. This is especially important when applying aerial imagery layers, since one want to avoid distorting the shape of e.g. square buildings. 
4. Meridians are equally spaced vertical lines. 
5. Rhumb lines are straight lines. A Rhumb line is the path you will travel while maintaining a constant bearing.

![](images/world.png)
Fig. Spherical Mercator projection - the world projected as a big square.

Since the Mercator projection goes to infinity at the poles, it doesn’t 
actually show the entire world. Using a square aspect ratio for the map, the 
maximum latitude shown is approximately 85.05°.

## Ground Resolution and Map Scale

In order to render the map, we also need to know the concepts of map scale and ground resolution. 
At the lowest level of detail (Level 0), the map is 512 x 512 pixels. 
At each successive level of detail, the map width and height doubles: Level 1 is 1024x1024 pixels, 
Level 2 is 2048x2048 pixels, Level 3 is 4096x4096 pixels, and so on. In general, the width and height of the map (in pixels) can be calculated as:

map width = map height = 512 * 2<sup>level</sup> pixels

The **ground resolution** is how long a single pixel is in meters in the real world, 
and it is measured in meters/pixel. The ground resolution varies depending on the level of detail 
and the latitude at which it is measured. 
The formula for calculating the ground resolution is as follows:

ground resolution = cos(latitude) * earth circumference / map width 


If we assume an earth circumference of 40075160 meters, the ground resolution can be calculated as:

= (cos(latitude) * 40075160 meters) / (512 * 2<sup>level</sup> pixels) 


The **map scale** is the ratio between map distance and ground distance,
when measured in the same units. For instance, at a map scale of 1:10000,
each meter on the map represents a ground distance of 10000 meters. Like 
the ground resolution, the map scale varies with the level of detail and the 
latitude of measurement. It can be calculated from the ground resolution as 
follows, given the screen resolution in dots per inch, typically 96 dpi:

map scale = 1:ground resolution * screen dpi / 0.0254 meters/inch 

= 1: (cos(latitude) * 40075160 * screen dpi) / (512 * 2<sup>level</sup> * 0.0254) 

This table shows each of these values at each level of detail, as measured at 
the Equator. (Note that the ground resolution and map scale also vary with the 
latitude, as shown in the equations above, but not shown in the table below.)


LEVEL OF DETAIL | MAP WIDTH AND HEIGHT (PIXELS) | GROUND RESOLUTION (METERS / PIXEL) | MAP SCALE (AT 96 DPI)
----------------|-------------------------------|------------------------------------|----------------------
0       |512	        | 78 271.5170	| 1:295 829 355.45
1	    |1 024	        | 39 135.7585	| 1:147 914 677.73
2	    |2 048	        | 19 567.8792	| 1:73 957 338.86
3	    |4 096	        | 9 783.9396	| 1:36 978 669.43
4	    |8 192	        | 4 891.9698	| 1:18 489 334.72
5	    |16 384	        | 2 445.9849	| 1:9 244 667.36
6	    |32 768	        | 1 222.9925	| 1:4 622 333.68
7	    |65 536	        | 611.4962	    | 1:2 311 166.84
8	    |131 072	    | 305.7481	    | 1:1 155 583.42
9	    |262 144	    | 152.8741	    | 1:577 791.71
10	    |524 288	    | 76.4370	    | 1:288 895.85
11	    |1 048 576	    | 38.2185	    | 1:144 447.93
12	    |2 097 152	    | 19.1093	    | 1:72 223.96
13	    |4 194 304	    | 9.5546	    | 1:36 111.98
14	    |8 388 608	    | 4.7773	    | 1:18 055.99
15	    |16 777 216	    | 2.3887	    | 1:9 028.00
16	    |33 554 432	    | 1.1943    	| 1:4 514.00
17	    |67 108 864	    | 0.5972	    | 1:2 257.00
18	    |134 217 728	| 0.2986	    | 1:1 128.50
19	    |268 435 456	| 0.1493	    | 1:564.25
20	    |536 870 912	| 0.0746	    | 1:282.12
21	    |1 073 741 824	| 0.0373	    | 1:141.06

## Pixel Coordinates 

Having chosen the projection and scale to use at each level of detail, we can
convert geographic coordinates into pixel coordinates. Since the map width and
height is different at each level, so are the pixel coordinates. The pixel at 
the upper-left corner of the map always has pixel coordinates (0, 0). The pixel
at the lower-right corner of the map has pixel coordinates (width-1, height-1)
or, equivalently, 
(512 * 2<sup>level</sup>-1, 512 * 2<sup>level</sup>-1). 
For example, at level 2, the pixel coordinates range from (0, 0) to (2047, 2047), 
like this:

![](images/world2.png)

Given latitude and longitude in degrees, and the level of detail, the pixel XY
coordinates can be calculated as follows:

pixelX = ((longitude + 180°) / 360°) * 512 * 2<sup>level</sup> 

pixelY = (0.5 - ln((1 + sin(latitude)) / (1 - sin(latitude))) / (4 * pi)) * 512 * 2<sup>level</sup> 

The latitude and longitude are assumed to be in the WGS 84 coordinate system.
The longitude is assumed to range from -180° to +180°, and the latitude
must be clipped to range from -85.05112878° to 85.05112878°.
This avoids a singularity at the poles, and it causes the projected map to be
square.

## Livemap Standard Coordinates

In most cases Livemap uses 30 bits to represent coordinates. This means that the upper-left (north-western) corner of the world has the coordinates (0,0) while the bottom-right (south-eastern) corner of the world has the coordinates (2<sup>30</sup>-1, 2<sup>30</sup>-1). This yields a ground resolution of 3.7 cm/pixel at the equator.

## Toxels and Polylines

A toxel is a geographical tile that also has a time dimension. Each day the
world is represented as a toxel that is 512x512x24h (pixels * pixels * time). A
toxel is always 512x512 pixels, but the depth in time may vary. 

![](images/toxel1.png)

A toxel consists of one or more trajectories. A trajectory is a vehicle’s
journey through time. A trajectory can be represented as a polyline
where each coordinate pair has a time attached to it, i.e. [{x<sub>0</sub>,y<sub>0</sub>,z<sub>0</sub>,t<sub>0</sub>}, …, {x<sub>n</sub>,y<sub>n</sub>,z<sub>n</sub>,t<sub>n</sub>}].

**A trajectory captures all relevant variables, such as geographical position, 
heading, speed, acceleration, departure time and arrival time.**

Consider the following example.

![](images/trajectories.png)

In this example there are three different trajectories.
1. A vehicle standing still for 24 hours.
2. A vehicle traveling from east to west in 24 hours.
3. A vehicle making 4 circles in 24 hours.

The above example has two spatial dimensions and one temporal dimension. However, the underlying data representation also has a z-coordinate, making toxel polylines 4D-objects. 
## Quad keys 

To optimize the performance of map retrieval and display, the rendered map is
cut into toxels, each of size 512 pixels x 512 pixels. As the number of pixels
differs at each level of detail, so does the number of toxels. Specifically, for a given level of detail, the whole world is represented by a grid of 2<sup>level</sup> x 2<sup>level</sup> toxels.

Each toxel is given a XY index ranging from (0, 0) in the upper left to (2<sup>level</sup>–1, 2<sup>level</sup>–1) in the lower right.
For example, at level 3 the toxel indexes range from (0, 0) to (7, 7) as
follows:

![](images/toxelxy.png)

Given a pair of pixel XY coordinates, you can easily determine the toxel XY
index containing that pixel:

toxelX = floor(pixelX / 512) 

toxelY = floor(pixelY / 512) 

To optimize the indexing and storage of toxels, the two-dimensional toxel XY
indexes are combined into one-dimensional strings called quadtree keys, or
"quadkeys" for short.

To convert toxel indices into a quadkey, the bits of the Y and X indices are
interleaved, and the result is interpreted as a base-4 number (without removing leading
zeros) and converted into a string. For instance, given toxel XY
indices of (3, 5) at level 3, the quadkey is determined as follows:

toxelX = 3 = 011<sub>2</sub> 

toxelY = 5 = 101<sub>2</sub>

quadkey = Y<sub>0</sub>X<sub>0</sub>Y<sub>1</sub>X<sub>1</sub>Y<sub>2</sub>X<sub>2</sub> = 100111<sub>2</sub> = 213<sub>4</sub> = "CBD"

**A=0<sub>4</sub>** | **B=1<sub>4</sub>**
----|----
**C=2<sub>4</sub>** | **D=3<sub>4</sub>**


Since the world-toxel (at level 0) would yield the empty string, all quadkeys 
are prefixed with the character 'T'. This means that toxel (3,5) at level 3 has the 
quadkey TCBD.

Quadkeys have several interesting properties. First, the length of a quadkey
(the number of characters) minus one (for the prefix) equals the level of 
detail of the corresponding toxel. Second, the quadkey of any toxel starts with
the quadkey of its parent toxel (the containing toxel at the previous level).
As shown in the example below, toxel T is the parent of toxels TA, TB, TC and
TD:

![](images/toxelabcd.png)

```js
// C# code to convert a quadkey in {x:y:level} format to {T[ABCD]*} format
public static string GetQuadkey(int x, int y, int level)
{
    var quadkey = new StringBuilder("T", level + 1);
    for (var i = level; i > 0; i--)
    {
        char digit = 'A';
        var mask = 1 << (i - 1);
        if ((x & mask) != 0) digit++;
        if ((y & mask) != 0) digit += (char)2;
        quadkey.Append(digit); 
    }
    return quadkey.ToString();
}
```

Finally, quadkeys provide a one-dimensional index key that usually preserves
the proximity of toxels in XY space. In other words, two toxels that have 
nearby XY indexes usually have quadkeys that are relatively close together. 
This is important for optimizing memory performance, because neighboring toxels
are usually requested in groups, and it’s desirable to keep those toxels in the
same memory area, in order to minimize the number of cache misses.

## Epoch keys

In analogy to quadkeys, to optimize the performance of map retrieval and 
display, the toxels are also sliced in time. Each quantified time slice is
assigned a key called an **epochkey**.

Epochkeys handles the temporal part of toxels. Consider the toxel in the figure
below. It covers all the trajectories in North America over 24 hours. It can
be split into two separate toxels which in turn cover 12 hours each. These 
toxels may in turn be split into 6 hour slices and so on.

![](images/epochkeysplit.png)

The initial epochkey that covers the whole day is defined as 1. Each temporal 
split will append a binary 0 for the first part, and a 1 for the later part. For instance, a
24-hour day of 86 400 000 milliseconds can be sliced into 1024 parts of 84375
milliseconds each, and each such part requires 11 binary digits.

Period    | Hex (Expochkey)   | Binary      | Time
----------|-------------------|-------------|--------------------------------------
24h       | 1                 | 1           | Whole day
12h       | 2                 | 10          | Midnight to noon (00:00 to 12:00)
12h       | 3                 | 11          | Noon to midnight (12:00 to 24:00)
6h        | 4                 | 100         | 00:00 to 06:00
6h        | 5                 | 101         | 06:00 to 12:00
6h        | 6                 | 110         | 12:00 to 18:00
6h        | 7                 | 111         | 18:00 to 24:00
3h        | 8                 | 1000        | 00:00 to 03:00
3h        | 9                 | 1001        | 03:00 to 06:00
3h        | A                 | 1010        | 06:00 to 09:00
3h        | B                 | 1011        | 09:00 to 12:00
3h        | C                 | 1100        | 12:00 to 15:00
3h        | D                 | 1101        | 15:00 to 18:00
3h        | E                 | 1110        | 18:00 to 21:00
3h        | F                 | 1111        | 21:00 to 24:00
90m       | 10                | 10000       | 00:00 to 01:30
...       |                   |             |
45m	      | 20                | 100000      | 00:00 to 00:45
...       |                   |             |
22m30s	  | 40	              | 1000000	    | 00:00:00 to 00:22:30
...       |                   |             |
11m15s	  | 80	              | 10000000    | 00:00:00 to 00:11:15
11m15s	  | 81	              | 10000001    | 00:11:15 to 00:22:30
11m15s	  | 82	              | 10000010    | 00:22:30 to 00:33:45
11m15s	  | 83	              | 10000011    | 00:33:45 to 00:45:00
...       |                   |             |
11m15s	  | FF	              | 11111111	| 23:48:45 to 24:00:00
...       |                   |             |
5m37.5s   | 100               | 100000000   | 00:00:00 to 00:05:37.5
...       |                   |             |
5m37.5s   | 1FF               | 111111111   | 23:54:22.5 to 24:00:00
...       |                   |             |
2m48.75s  | 200               | 1000000000  | 00:00:00 to 00:02:48.75
...       |                   |             |
2m48.75s  | 3FF               | 1111111111  | 23:57:11.25 to 24:00:00
...       |                   |             |
1m24.375s | 400               | 10000000000 | 00:00:00 to 00:01:24.375
...       |                   |             |
1m24.375s | 7FF               | 11111111111 | 23:58:35.625 to 24:00:00

```csharp
// C# code to convert epoch-code to start-/stop time
public static Tuple<Time,Time> TimeSlice(int code)
{
        int mask = 0;
        int length = 86400000; //Milliseconds per day
        for(int i = code; i > 1; i >>= 1)
        {
                mask <<= 1;
                mask |= 1;
                length >>= 1;
        }
        var begin = new Time((code & mask) * length);
        var end = begin.AddMillisecods(length);
        return new Tuple<Time,Time>(begin, end);
}

```

# Livemap API

The Livemap Cloud Platform has several APIs for fetching and updating data, but in this section we will focus on the GetToxel-API.

## Notational Conventions

NOTATION            | MEANING       | EXAMPLE
--------------------|---------------|---------------------
<nobr>CURLY BRACKETS { }</nobr>  | Required item | api.livemap24.com/v1/trips/\{tripId}.json<br>*The tripId identifier is required.*
<nobr>SQUARE BRACKETS [ ]</nobr> | Optional item | api.livemap24.com/v1/trips/42.json?[callback=foobar]<br>*Specifying a JSONP callback function is optional.*

## File Extensions and MIME-types

File extension  | MIME-type                 | Description
----------------|---------------------------|--------------------------------
.xml	        | application/xml	        | Compact XML format
.text.xml	    | application/xml	        | Human readable (long names and indentation) XML format
.json	        | application/json	        | Compact JSON format
.text.json	    | application/json	        | Human readable (long names and indentation) JSON format
.bin	        | application/octet-stream  | Veridict proprietary binary format

The compact formats use short names in properties in order to save bandwidth, e.g. &lt;Toxel&gt; becomes &lt;txl&gt;. 
This is the recommended format for production releases. Only the human readable format is described below, 
the compact name is written at the end of each line as a comment.

## API key

To use Livemaps APIs, you need to obtain an API key. 

Once you have a key, you can pass it to any API using the key URL parameter:

https://beta.livemap24.com/v3/toxels/20160428/512/tbcacacab-3090.bin&key=YOUR_API_KEY

For now, you can use demo key **46CE45692D9645CA3DCD945353D0E65D**. The demo key may change at any moment without notice.

## GetToxel API

Use this API to fetch a toxel in XML, JSON or binary format.

Uri template:
https://beta.livemap24.com/v3/toxels/{date}/{toxelSize}/{quadkey}-{epochkey}.{format}?key={key}[&fields={fields}]

Parameter      | Description
---------------|------------
\{date}        | **string (required)** Toxel date UTC as YYYYMMDD or YYYY-MM-DD. Example: 20160614
\{toxelSize}   | **string (required)** Toxel size. Must be 512.
\{quadkey}     | **string (required)** The spatial part of the toxel identifier. Case insensitive. See Quad keys. Example: tbababab
\{epochkey}    | **string (required)** The temporal part of the toxel identifier. Case insensitive. See Epoch keys. Example: 3f
\{key}         | **string (required)** API-key. Example: 46CE45692D9645CA3DCD945353D0E65D
\{format}      | **string (required)** File format. See File Extensions and MIME-types. One of: xml text.xml json text.json bin
\{fields}      | **integer (optional)** Data fields to include in response. See souce code (enum ToxelFields).

Example:

https://beta.livemap24.com/v3/toxels/20170914/512/tbcaadbccbda-3090.json?fields=17

---------

Copyright © 2017 Veridict
All rights reserved
