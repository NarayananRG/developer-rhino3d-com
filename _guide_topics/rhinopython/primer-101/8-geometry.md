---
title: 8 Geometry
description:
authors: ['Skylar Tibbits', 'Arthur van der Harten', 'Steve Baer']
author_contacts: ['steve']
apis: ['RhinoPython']
languages: ['Python']
platforms: ['Windows', 'Mac']
categories: ['Python Primer']
origin:
order: 15
keywords: ['python', 'commands']
layout: toc-guide-page
---

## 8.1 The openNURBS™ kernel

Now that you are familiar with the basics of scripting, it is time to start with the actual geometry part of RhinoScript. To keep things interesting we've used plenty of Rhino methods in examples before now, but that was all peanuts. Now you will embark upon that great journey which, if you survive, will turn you into a real 3D geek.

As already mentioned in Chapter 3, Rhinoceros is built upon the openNURBS™ kernel which supplies the bulk of the geometry and file I/O functions. All plugins that deal with geometry tap into this rich resource and the RhinoScript plugin is no exception. Although Rhino is marketed as a "NURBS modeler", it does have a basic understanding of other types of geometry as well. Some of these are available to the general Rhino user, others are only available to programmers. When writting in Python you will not be dealing directly with any 
openNURBS™ code since RhinoScript wraps it all up into an easy-to-swallow package. However, programmers need to have a much higher level of comprehension than users which is why we'll dig fairly deep.

## 8.2 Objects in Rhino

All objects in Rhino are composed of a geometry part and an attribute part. There are quite a few different geometry types but the attributes always follow the same format. The attributes store information such as object name, color, layer, isocurve density, linetype and so on. Not all attributes make sense for all geometry types, points for example do not use linetypes or materials but they are capable of storing this information nevertheless. Most attributes and properties are fairly straightforward and can be read and assigned to objects at will. 

<img src="{{ site.baseurl }}/images/primer-rhinoobjects.svg" width="90%" float="right">

This table lists most of the attributes and properties which are available to plugin developers. Most of these have been wrapped in the RhinoScript plugin, others are missing at this point in time and the custom user data element is special. We'll get to user data after we're done with the basic geometry chapters.

The following procedure displays some attributes of a single object in a dialog box. There is nothing exciting going on here so I'll refrain from providing a step-by-step explanation.

```python
import rhinoscriptsyntax as rs

def displayobjectattributes(object_id):
    source = "By Layer", "By Object", "By Parent"
    data = []
    data.append( "Object attributes for :"+str(object_id) )
    data.append( "Description: " + rs.ObjectDescription(object_id))
    data.append( "Layer: " + rs.ObjectLayer(object_id))
    #data.append( "LineType: " + rs.ObjectLineType(object_id))
    #data.append( "LineTypeSource: " + rs.ObjectLineTypeSource(object_id))
    data.append( "MaterialSource: " + str(rs.ObjectMaterialSource(object_id)))
    
    name = rs.ObjectName(object_id)
    if not name: data.append("<Unnamed object>")
    else: data.append("Name: " + name)
    
    groups = rs.ObjectGroups(object_id)
    if groups:
        for i,group in enumerate(groups):
            data.append( "Group(%d): %s" % i+1, group )
    else:
        data.append("<Ungrouped object>")

    s = ""
    for line in data: s += line + "\n"
    rs.EditBox(s, "Object attributes", "RhinoPython")


if __name__=="__main__":
    id = rs.GetObject()
    displayobjectattributes(id)
```

<img src="{{ site.baseurl }}/images/primer-objectattributedialog.png" width="45%" float="right">

## 8.3 Points and Pointclouds

Everything begins with points. A point is nothing more than a list of values called a coordinate. The number of values in the list corresponds with the number of dimensions of the space it resides in. Space is usually denoted with an R and a superscript value indicating the number of dimensions. (The 'R' stems from the world 'real' which means the space is continuous. We should keep in mind that a digital representation always has gaps, even though we are rarely confronted with them.) 

Points in 3D space, or R3 thus have three coordinates, usually referred to as [x,y,z]. Points in R2 have only two coordinates which are either called [x,y] or [u,v] depending on what kind of two dimensional space we're talking about. Points in R1 are denoted with a single value. Although we tend not to think of one-dimensional points as 'points', there is no mathematical difference; the same rules apply. One-dimensional points are often referred to as 'parameters' and we denote them with [t] or [p].

<img src="{{ site.baseurl }}/images/primer-rhinospaces.svg" width="90%" float="right">

The image on the left shows the R3 world space, it is continuous and infinite. The x-coordinate of a point in this space is the projection (the red dotted line) of that point onto the x-axis (the red solid line). Points are always specified in world coordinates in Rhino.

R2 world space (not drawn) is the same as R3 world space, except that it lacks a z-component. It is still continuous
and infinite. R2 parameter space however is bound to a finite surface as shown in the center image. It is still continuous, I.e. hypothetically there is an infinite amount of points on the surface, but the maximum distance between any of these points is very much limited. R2 parameter coordinates are only valid if they do not exceed a certain range. In the example drawing the range has been set between 0.0 and 1.0 for both [u] and [v]directions, but it could be any finite domain. A point with coordinates [1.5, 0.6] would be somewhere outside the surface and thus invalid.

Since the surface which defines this particular parameter space resides in regular R3 world space, we can always translate a parametric coordinate into a 3d world coordinate. The point [0.2, 0.4] on the surface for example is the same as point [1.8, 2.0, 4.1] in world coordinates. Once we transform or deform the surface, the R3 coordinates which correspond with [0.2, 0.4] will change. Note that the opposite is not true, we can translate any R2 parameter coordinate into a 3D world coordinate, but there are many 3D world coordinates that are not on the surface and which can therefore not be written as an R2 parameter coordinate. However, we can always project a 3D world coordinate onto the surface using the closest-point relationship. We'll discuss this in more detail later on.

If the above is a hard concept to swallow, it might help you to think of yourself and your position in space. We usually tend to use local coordinate systems to describe our whereabouts; "I'm sitting in the third seat on the seventh row in the movie theatre", "I live in apartment 24 on the fifth floor", "I'm in the back seat". Some of these are variations to the global coordinate system (latitude, longitude, elevation), while others use a different anchor point. If the car you're in is on the road, your position in global coordinates is changing all the time, even though you remain in the same back seat 'coordinate'.

Let's start with conversion from R1 to R3 space. The following script will add 500 colored points to the 
document, all of which are sampled at regular intervals across the R1 parameter space of a curve object:

```python
import rhinoscriptsyntax as rs

def main():
    curve_id = rs.GetObject("Select a curve to sample", 4, True, True)
    if not curve_id: return
    
    rs.EnableRedraw(False)
    t = 0
    while t<=1.0:
        addpointat_r1_parameter(curve_id,t)
        t+=0.002
    rs.EnableRedraw(True)

def addpointat_r1_parameter(curve_id, parameter):
    domain = rs.CurveDomain(curve_id)
    
    
    r1_param = domain[0] + parameter*(domain[1]-domain[0])
    r3point = rs.EvaluateCurve(curve_id, r1_param)
    if r3point:
        point_id = rs.AddPoint(r3point)
        rs.ObjectColor(point_id, parametercolor(parameter))

def parametercolor(parameter):
    red = 255 * parameter
    if red<0: red=0
    if red>255: red=255
    return (red,0,255-red)

if __name__=="__main__":
    main()
```

<img src="{{ site.baseurl }}/images/primer-curveparameterspace.svg" width="45%" float="right">

For no good reason whatsoever, we'll start with the bottom most function:

<table rules="rows">
<tr>
<th>Line</th>	
<th>Description</th>
</tr>
<tr>
<td style="vertical-align:top;text-align:right;padding:0px 10px;">24</td>
<td>Standard out-of-the-box function declaration which takes a single double value. This function is 
supposed to return a colour which changes gradually from blue to red as parameter changes from zero to one. Values outside of the range {0.0~1.0} will be clipped.</td>
</tr>
<tr>
<td style="vertical-align:top;text-align:right;padding:0px 10px;">25</td>
<td>The red component of the colour we're going to return is declared here and assigned the naive value of 255 times the parameter. Colour components must have a value between and including 0 and 255. If we attempt to construct a colour with lower or higher values a run-time error will spoil the party.</td>
</tr>
<tr>
<td style="vertical-align:top;text-align:right;padding:0px 10px;">26...27</td>
<td>Here's where we make sure the party can continue unimpeded.</td>
</tr>
<tr>
<td style="vertical-align:top;text-align:right;padding:0px 10px;">28</td>
<td>Compute the colour gradient value. If parameter equals zero we want blue (0,0,255) and if it equals one we want red (255,0,0). So the green component is always zero while blue and red see-saw between 0 and 255.</td>
</tr>
</table>

Now, on to function *AddPointAtR1Parameter()*. As the name implies, this function will add a single point in 3D world space based on the parameter coordinate of a curve object. In order to work correctly this function must know what curve we're talking about and what parameter we want to sample. Instead of passing the actual parameter which is bound to the curve domain (and could be anything) we're passing a unitized one. 
I.e. we pretend the curve domain is between zero and one. This function will have to wrap the required math for translating unitized parameters into actual parameters.

Since we're calling this function a lot (once for every point we want to add), it is actually a bit odd to put all the heavy-duty stuff inside it. We only really need to perform the overhead costs of 'unitized parameter + actual parameter' calculation once, so it makes more sense to put it in a higher level function. Still, it will be very quick so there's no need to optimize it yet.


<table rules="rows">
<tr>
<th>Line</th>	
<th>Description</th>
</tr>
<tr>
<td style="vertical-align:top;text-align:right;padding:0px 10px;">14</td>
<td>Function declaration.</td>
</tr>
<tr>
<td style="vertical-align:top;text-align:right;padding:0px 10px;">15...16</td>
<td>Get the curve domain and check for <i>Null</i>. It will be <i>Null</i> if the ID does not represent a proper curve object. The <i>Rhino.CurveDomain()</i> method will return an array of two doubles which indicate the minimum and maximum t-parameters which lie on the curve.</td>
</tr>
<tr>
<td style="vertical-align:top;text-align:right;padding:0px 10px;">18</td>
<td>Translate the unitized R1 coordinate into actual domain coordinates.</td>
</tr>
<tr>
<td style="vertical-align:top;text-align:right;padding:0px 10px;">19</td>
<td>Evaluate the curve at the specified parameter. Rhino.EvaluateCurve() takes an R1 coordinate and returns an R3 coordinate.</td>
</tr>
<tr>
<td style="vertical-align:top;text-align:right;padding:0px 10px;">21</td>
<td>Add the point, it will have default attributes.</td>
</tr>
<tr>
<td style="vertical-align:top;text-align:right;padding:0px 10px;">22</td>
<td>Set the custom colour. This will automatically change the color-source attribute to By Object.</td>
</tr>
</table>

The distribution of R1 points on a spiral is not very enticing since it approximates a division by equal length segments in R3 space. When we run the same script on less regular curves it becomes easier to grasp what parameter space is all about:

<img src="{{ site.baseurl }}/images/primer-curvestructure.svg" width="100%" float="right">

Let's take a look at an example which uses all parameter spaces we've discussed so far:

```python
import rhinoscriptsyntax as rs

def main():
    surface_id = rs.GetObject("Select a surface to sample", 8, True)
    if not surface_id: return

    curve_id = rs.GetObject("Select a curve to measure", 4, True, True)
    if not curve_id: return

    points = rs.DivideCurve(curve_id, 500)
    rs.EnableRedraw(False)
    for point in points: evaluatedeviation(surface_id, 1.0, point)
    rs.EnableRedraw(True)

def evaluatedeviation( surface_id, threshold, sample ):
    r2point = rs.SurfaceClosestPoint(surface_id, sample)
    if not r2point: return

    r3point = rs.EvaluateSurface(surface_id, r2point[0], r2point[1])
    if not r3point: return

    deviation = rs.Distance(r3point, sample)
    if deviation<=threshold: return

    rs.AddPoint(sample)
    rs.AddLine(sample, r3point)

if __name__=="__main__":
    main()
```

<table>
<tr>
<td>
This script will compare a bunch of points on a curve to their projection on a surface. If the distance exceeds one unit, a line and a point will be added.
<br><br>
First, the R1 points are translated into R3 coordinates so we can 
project them onto the surface, getting the R2 coordinate [u,v] in return. This R2 point has to be translated into R3 space as well, since we need to know the distance between the R1 point on the curve and the R2 point on the surface. Distances can only be measured if both points reside in the same number of dimensions, so we need to translate them into R3 as well.
<br>
Told you it was a piece of cake...
</td>
<td width="30%"><img src="{{ site.baseurl }}/images/primer-surfaceparameterspace.svg" width="100%" height="300" float="right"></td>
</tr>
</table>

<table rules="rows">
<tr>
<th>Line</th>	
<th>Description</th>
</tr>
<tr>
<td style="vertical-align:top;text-align:right;padding:0px 10px;">10</td>
<td>We're using the <i>Rhino.DivideCurve()</i> method to get all the R3 coordinates on the curve in one go. This saves us a lot of looping and evaluating.</td>
</tr>
<tr>
<td style="vertical-align:top;text-align:right;padding:0px 10px;">24</td>
<td><i>Rhino.SurfaceClosestPoint()</i> returns an array of two doubles representing the R2 point on the surface (in {u,v} coordinates) which is closest to the sample point.</td>
</tr>
<tr>
<td style="vertical-align:top;text-align:right;padding:0px 10px;">27</td>
<td>Rhino.EvaluateSurface() in turn translates the R2 parameter coordinate into R3 world coordinates</td>
</tr>
<tr>
<td style="vertical-align:top;text-align:right;padding:0px 10px;">30...38</td>
<td>Compute the distance between the two points and add geometry if necessary. This function returns True if the deviation is less than one unit, False if it is more than one unit and Null if something went wrong.</td>
</tr>
</table>

One more time just for kicks. We project the R1 parameter coordinate on the curve into 3D space (Step A), then we project that R3 coordinate onto the surface getting the R2 coordinate of the closest point (Step B). We evaluate the surface at R2, getting the R3 coordinate in 3D world space (Step C), and we finally measure the distance between the two R3 points to determine the deviation:


<img src="{{ site.baseurl }}/images/primer-surfaceparameterspacediagram.svg" width="60%" float="right">

## 8.4 Lines and Polylines

You'll be glad to learn that (poly)lines are essentially the same as point-lists. The only difference is that we treat the points as a series rather than an anonymous collection, which enables us to draw lines between them. There is some nasty stuff going on which might cause problems down the road so perhaps it's best to get it over with quick.

There are several ways in which polylines can be manifested in openNURBS™ and thus in Rhino. There is a special polyline class which is simply a list of ordered points. It has no overhead data so this is the simplest case. It's also possible for regular nurbs curves to behave as polylines when they have their degree set to 1. In 
addition, a polyline could also be a polycurve made up of line segments, polyline segments, degree=1 nurbs curves or a combination of the above. If you create a polyline using the _Polyline command, you will get a proper polyline object as the Object Properties Details dialog on the left shows:

The dialog claims an "Open polyline with 8 points". However, when we drag a control-point Rhino will 
automatically convert any curve to a Nurbs curve, as the image on the right shows. It is now an open nurbs curve of degree=1. From a geometric point of view, these two curves are identical. From a programmatic point of view, they are anything but. For the time being we will only deal with 'proper' polylines though; lists of sequential coordinates. For purposes of clarification I've added two example functions which perform basic operations on polyline point-lists.

Compute the length of a polyline point-array:

```python
def PolylineLength(arrVertices):
    PolylineLength = 0.0
    for i in range(0,len(arrVertices)-1):
        PolylineLength = PolylineLength + rs.Distance(arrVertices[i], arrVertices[i+1])
```

Subdivide a polyline by adding extra vertices halfway between all existing vertices:

```python
def SubDividePolyline(arrV)
    arrSubD = []

    for i in range(0, len(arrV)-1):
        'copy the original vertex location
        arrSubD.append(arrV[i])
        'compute the average of the current vertex and the next one
        arrSubD.append([arrV[i][0] + arrV[i+1][0]] / 2.0, _
                                    [arrV(i][1] + arrV[i+1][1]] / 2.0, _
                                    [arrV[i][2] + arrV[i+1][2]] / 2.0])

    'copy the last vertex (this is skipped by the loop)
    arrSubD.append(arrV[len(arrV)])
    return arrSubD
```

<img src="{{ site.baseurl }}/images/primer-polylinetonurbsdragchange.png" width="90%" float="right">

<table>
<tr>
<td>
No rocket science yet, but brace yourself for the next bit...
<br><br>
As you know, the shortest path between two points is a straight line. This is true for all our space definitions, from R1 to RN. However, the shortest path in R2 space is not necessarily the same shortest path in R3 space. If we want to connect two points on a surface with a straight line in R2, all we need to do is plot a linear course through the surface [u,v] space. (Since we can only add curves to Rhino which use 3D world coordinates, we'll need a fair amount of samples to give the impression of smoothness.) The thick red curve in the 
adjacent illustration is the shortest path in R2 parameter 
space connecting [A] and [B]. We can clearly see that this is definitely not the shortest path in R3 space.
</td>
<td width="30%"><img src="{{ site.baseurl }}/images/primer-r2shortpath.svg" width="100%" height="300" float="right"></td>
</tr>
</table>

We can clearly see this because we're used to things happening in R3 space, which is why this whole R2/R3 thing is so thoroughly counter intuitive to begin with. The green, dotted curve is the actual shortest path in R3 space which still respects the limitation of the surface (I.e. it can be projected onto the surface without any loss of information). The following function was used to create the red curve; it creates a polyline which represents the shortest path from [A] to [B] in surface parameter space:

```python
def getr2pathonsurface(surface_id, segments, prompt1, prompt2):
    start_point = rs.GetPointOnSurface(surface_id, prompt1)
    if not start_point: return

    end_point = rs.GetPointOnSurface(surface_id, prompt2)
    if not end_point: return

    if rs.Distance(start_point, end_point)==0.0: return

    uva = rs.SurfaceClosestPoint(surface_id, start_point)
    uvb = rs.SurfaceClosestPoint(surface_id, end_point)

    path = []
    for i in range(segments):
        t = i / segments
        u = uva[0] + t*(uvb[0] - uva[0])
        v = uva[1] + t*(uvb[1] - uva[1])
        pt = rs.EvaluateSurface(surface_id, u, v)
        path.append(pt)
    return path
```

<table rules="rows">
<tr>
<th>Line</th>	
<th>Description</th>
</tr>
<tr>
<td style="vertical-align:top;text-align:right;padding:0px 10px;">1</td>
<td>This function takes four arguments; the ID of the surface onto which to plot the shortest route, the number of segments for the path polyline and the prompts to use for picking the A and B point.</td>
</tr>
<tr>
<td style="vertical-align:top;text-align:right;padding:0px 10px;">1...3</td>
<td>Prompt the user for the {A} point on the surface. Return if the user does not enter a point.</td>
</tr>
<tr>
<td style="vertical-align:top;text-align:right;padding:0px 10px;">5...6</td>
<td>Prompt the user for the {B} point on the surface. Return if the user does not enter a point.</td>
</tr>
<tr>
<td style="vertical-align:top;text-align:right;padding:0px 10px;">10...11</td>
<td>Project {A} and {B} onto the surface to get the respective R2 coordinates <i>uva</i> and <i>uvb</i>.</td>
</tr>
<tr>
<td style="vertical-align:top;text-align:right;padding:0px 10px;">13</td>
<td>Declare the list which is going to store all the polyline vertices.</td>
</tr>
<tr>
<td style="vertical-align:top;text-align:right;padding:0px 10px;">14</td>
<td>Since this algorithm is segment-based, we know in advance how many vertices the polyline will have and thus how often we will have to sample the surface.</td>
</tr>
<tr>
<td style="vertical-align:top;text-align:right;padding:0px 10px;">15</td>
<td><i>t</i> is a value which ranges from 0.0 to 1.0 over the course of our loop</td>
</tr>
<tr>
<td style="vertical-align:top;text-align:right;padding:0px 10px;">16...17</td>
<td>Use the current value of <i>t</i> to sample the surface somewhere in between <i>uvA</i> and <i>uvB</i>.</td>
</tr>
<tr>
<td style="vertical-align:top;text-align:right;padding:0px 10px;">18</td>
<td><i>rs.EvaluateSurface()</i> takes a {u} and a {v} value and spits out a 3D-world coordinate. This is just a friendly way of saying that it converts from R2 to R3.</td>
</tr>
</table>

We're going to combine the previous examples in order to make a real geodesic path routine in Rhino. This is a fairly complex algorithm and I'll do my best to explain to you how it works before we get into any actual code.

First we'll create a polyline which describes the shortest path between [A] and [B] in R2 space. This is our base curve. It will be a very coarse approximation, only ten segments in total. We'll create it using the function on page 54. Unfortunately that function does not take closed surfaces into account. In the paragraph on nurbs surfaces we'll elaborate on this.

Once we've got our base shape we'll enter the iterative part. The iteration consists of two nested loops, which we will put in two different functions in order to avoid too much nesting and indenting. We're going to write four functions in addition to the ones already discussed in this paragraph:

- The main geodesic routine
- ProjectPolyline()
- SmoothPolyline()
- GeodesicFit()

The purpose of the main routine is the same as always; to collect the initial data and make sure the script completes as successfully as possible. Since we're going to calculate the geodesic curve between two points on a surface, the initial data consists only of a surface ID and two points in surface parameter space. The algorithm for finding the geodesic curve is a relatively slow one and it is not very good at making major changes to dense polylines. That is why we will be feeding it the problem in bite-size chunks. It is because of this reason that our initial base curve (the first bite) will only have ten segments. We'll compute the geodesic path for these ten segments, then subdivide the curve into twenty segments and recompute the geodesic, then subdivide into 40 and so on and so forth until further subdivision no longer results in a shorter overall curve.

The *ProjectPolyline()* function will be responsible for making sure all the vertices of a polyline point-array are in fact coincident with a certain surface. In order to do this it must project the R3 coordinates of the polyline onto the surface, and then again evaluate that projection back into R3 space. This is called 'pulling'.

The purpose of *SmoothPolyline()* will be to average all polyline vertices with their neighbours. This function will be very similar to our previous example, except it will be much simpler since we know for a fact we're not dealing with nurbs curves here. We do not need to worry about knots, weights, degrees and domains.

*GeodesicFit()* is the essential geodesic routine. We expect it to deform any given polyline into the best possible geodesic curve, no matter how coarse and wrong the input is. The algorithm in question is a very naive solution to the geodesic problem and it will run much slower than Rhinos native _ShortPath command. The upside is that our script, once finished, will be able to deal with self-intersecting surfaces.

The underlying theory of this algorithm is synonymous with the simulation of a contracting rubber band, with the one difference that our rubber band is not allowed to leave the surface. The process is iterative and though we expect every iteration to yield a certain improvement over the last one, the amount of improvement will diminish as we near the ideal solution. Once we feel the improvement has become negligible we'll abort the function.

In order to simulate a rubber band we require two steps; smoothing and projecting. First we allow the rubber band to contract (it always wants to contract into a straight line between [A] and [B]). This contraction happens in R3 space which means the vertices of the polyline will probably end up away from the surface. We must then re-impose these surface constraints. These two operations have been hoisted into functions #2 and #3.


<img src="{{ site.baseurl }}/images/primer-geodesiccurvediagram.svg" width="65%" float="right">

The illustration depicts the two steps which compose a single iteration of the geodesic routine. The black polyline is projected onto the surface giving the red polyline. The red curve in turn is smoothed into the green  curve. Note that the actual algorithm performs these two steps in the reverse order; smoothing first, projection second.

We'll start with the simplest function:

```python
def projectpolyline(vertices, surface_id):
    polyline = []
    for vertex in vertices:
        pt = rs.BrepClosestPoint(surface_id, vertex)
        if pt: polyline.append(pt[0])
    return polyline
```

<table rules="rows">
<tr>
<th>Line</th>	
<th>Description</th>
</tr>
<tr>
<td style="vertical-align:top;text-align:right;padding:0px 10px;">1...3</td>
<td>Since this is a specialized def which we will only be using inside this script, we can skip projecting the first and last point. We can safely assume the polyline is open and that both endpoints will already be on the curve.</td>
</tr>
<tr>
<td style="vertical-align:top;text-align:right;padding:0px 10px;">4</td>
<td>We ask Rhino for the closest point on the surface object given our polyline vertex coordinate. The reason why we do not use <i>rs.SurfaceClosestPoint()</i> is because <i>BRepClosestPoint()</i> takes trims into account. This is a nice bonus we can get for free. The native <i>_ShortPath</i> command does not deal with trims at all. We are of course not interested in aping something which already exists, we want to make something better.</td>
</tr>
<tr>
<td style="vertical-align:top;text-align:right;padding:0px 10px;">5</td>
<td>If <i>BRepClosestPoint()</i> returned Null something went wrong after all. We cannot project the vertex in this case so we'll simply ignore it. We could of course short-circuit the whole operation after a failure like this, but I prefer to press on and see what comes out the other end. </td>
</tr>
<tr>
<td style="vertical-align:top;text-align:right;padding:0px 10px;">6</td>
<td>The <i>BRepClosestPoint()</i> method returns a lot of information, not just the R2 coordinate. In fact it returns a tuple of data, the first element of which is the R3 closest point. This means we do not have to translate the uv coordinate into xyz ourselves. Huzzah! Assign it to the vertex and move on.</td>
</tr>
</table>

```python
def smoothpolyline(vertices):
    smooth = []
    smooth.append(vertices[0])

    for i in range(1, len(vertices)-1):
        prev = vertices[i-1]
        this = vertices[i]
        next = vertices[i+1]
        pt = (prev+this+next) / 3.0
        smooth.append(pt)
    smooth.append(vertices[len(vertices)-1])
    return smooth
```


<table rules="rows">
<tr>
<th>Line</th>	
<th>Description</th>
</tr>
<tr>
<td style="vertical-align:top;text-align:right;padding:0px 10px;">1...3 6...8</td>
<td>Since we need the original coordinates throughout the smoothing operation we cannot deform it 
directly. That is why we need to make a copy of each vertex point before we start messing about with coordinates.</td>
</tr>
<tr>
<td style="vertical-align:top;text-align:right;padding:0px 10px;">9</td>
<td>What we do here is average the x, y and z coordinates of the current vertex ('current' as defined by i) using both itself and its neighbours.
<br><br>
We iterate through all the internal vertices and add the Point3d objects together, rather than explicitly adding their x, y and z components together. Writing smaller functions will not make the code go faster, but it does mean we just get to write less junk. Also, it means adjustments are easier to make afterwards since less code-rewriting is required.</td>
</tr>
</table>  

Time for the bit that sounded so difficult on the previous page, the actual geodesic curve fitter routine:

```python
def geodesicfit(vertices, surface_id, tolerance):
    length = polylinelength(vertices)
    while True:
        vertices = smoothpolyline(vertices)
        vertices = projectpolyline(vertices, surface_id)
        newlength = polylinelength(vertices)
        if abs(newlength-length)<tolerance: return vertices
        length = newlength
```

<table rules="rows">
<tr>
<th>Line</th>	
<th>Description</th>
</tr>
<tr>
<td style="vertical-align:top;text-align:right;padding:0px 10px;">1</td>
<td>Hah... that doesn't look so bad after all, does it? You'll notice that it's often the stuff which is easy to explain that ends up taking a lot of lines of code. Rigid mathematical and logical structures can typically be coded very efficiently.</td>
</tr>
<tr>
<td style="vertical-align:top;text-align:right;padding:0px 10px;">2</td>
<td>We'll be monitoring the progress of each iteration and once the curve no longer becomes noticeably shorter (where 'noticeable' is defined by the <i>tolerance</i> argument), we'll call the 'intermediate result' the 'final result' and return execution to the caller. In order to monitor this progress, we need to remember how long the curve was before we started; <i>length</i> is created for this purpose.</td>
</tr>
<tr>
<td style="vertical-align:top;text-align:right;padding:0px 10px;">3</td>
<td>Whenever you see a while True: without any standard escape clause you should be on your toes. This is potentially an infinite loop. I have tested it rather thoroughly and have been unable to make it run more than 120 times. Experimental data is never watertight proof, the routine could theoretically fall into a stable state where it jumps between two solutions. If this happens, the loop will run forever.
<br><br>
You are of course welcome to add additional escape clauses if you deem that necessary.</td>
</tr>
<tr>
<td style="vertical-align:top;text-align:right;padding:0px 10px;">4...5</td>
<td>Place the calls to the functions on page 56. These are the bones of the algorithm.</td>
</tr>
<tr>
<td style="vertical-align:top;text-align:right;padding:0px 10px;">6</td>
<td>Compute the new length of the polyline.</td>
</tr>
<tr>
<td style="vertical-align:top;text-align:right;padding:0px 10px;">7</td>
<td>Check to see whether or not it is worth carrying on.</td>
</tr>
<tr>
<td style="vertical-align:top;text-align:right;padding:0px 10px;">8</td>
<td>Apparently it was, we need now to remember this new length as our frame of reference.</td>
</tr>
</table>

The main subroutine takes some explaining. It performs a lot of different tasks which always makes a block of code harder to read. It would have been better to split it up into more discrete chunks, but we're already using seven different functions for this script and I feel we are nearing the ceiling. Remember that splitting problems into smaller parts is a good way to organize your thoughts, but it doesn't actually solve anything. You'll need to find a good balance between splitting and lumping.

```python
def geodesiccurve():
    surface_id = rs.GetObject("Select surface for geodesic curve solution", 8, True, True)
    if not surface_id: return

    vertices = getr2pathonsurface(surface_id, 10, "Start of geodes curve", "End of geodes curve")
    if not vertices: return

    tolerance = rs.UnitAbsoluteTolerance() / 10
    length = 1e300
    newlength = 0.0

    while True:
        print("Solving geodesic fit for %d samples" % len(vertices))
        vertices = geodesicfit(vertices, surface_id, tolerance)

        newlength = polylinelength(vertices)
        if abs(newlength-length)<tolerance: break
        if len(vertices)>1000: break
        vertices = subdividepolyline(vertices)
        length = newlength

    rs.AddPolyline(vertices)
    print "Geodesic curve added with length: ", newlength
```


<table rules="rows">
<tr>
<th>Line</th>	
<th>Description</th>
</tr>
<tr>
<td style="vertical-align:top;text-align:right;padding:0px 10px;">2...3</td>
<td>Get the surface to be used in the geodesic routine.</td>
</tr>
<tr>
<td style="vertical-align:top;text-align:right;padding:0px 10px;">5...6</td>
<td>Declare a variable which will store the polyline vertices. Since the return value of <i>getr2pathonsurface()</i> is already a list, we do not need to declare it with empty brackets - '[ ]' - as we have done elsewhere.</td>
</tr>
<tr>
<td style="vertical-align:top;text-align:right;padding:0px 10px;">8</td>
<td>The tolerance used in our script will be 10% of the absolute tolerance of the document.</td>
</tr>
<tr>
<td style="vertical-align:top;text-align:right;padding:0px 10px;">9...12</td>
<td>This loop also uses a length comparison in order to determine whether or not to continue. But instead of evaluating the length of a polyline before and after a smooth/project iteration, it measures the difference before and after a subdivide/geodesicfit iteration. The goal of this evaluation is to decide whether or not further elaboration will pay off. The variables <i>length</i> and <i>newlength</i> are used in the same context as on the previous page.</td>
</tr>
<tr>
<td style="vertical-align:top;text-align:right;padding:0px 10px;">13</td>
<td>Display a message in the command-line informing the user about the progress we're making. This script may run for quite some time so it's important not to let the user think the damn thing has crashed.</td>
</tr>
<tr>
<td style="vertical-align:top;text-align:right;padding:0px 10px;">14</td>
<td>Place a call to the <i>GeodesicFit()</i> subroutine.</td>
</tr>
<tr>
<td style="vertical-align:top;text-align:right;padding:0px 10px;">16...17</td>
<td>Compare the improvement in length, exit the loop when there's no progress of any value.</td>
</tr>
<tr>
<td style="vertical-align:top;text-align:right;padding:0px 10px;">18</td>
<td>A safety-switch. We don't want our curve to become too dense.</td>
</tr>
<tr>
<td style="vertical-align:top;text-align:right;padding:0px 10px;">19</td>
<td>A call to <i>subdividepolyline()</i> will double the amount of vertices in the polyline. The newly added vertices will not be on the surface, so we must make sure to call <i>geodesicfit()</i> at least once before we add this new polyline to the document.</td>
</tr>
<tr>
<td style="vertical-align:top;text-align:right;padding:0px 10px;">22...23</td>
<td>Add the curve and print a message about the length.</td>
</tr>
</table>

## 8.5 Planes

Planes are not genuine objects in Rhino, they are used to define a coordinate system in 3D world space. In fact, it's best to think of planes as vectors, they are merely mathematical constructs. Although planes are internally defined by a parametric equation, I find it easiest to think of them as a set of axes:

<img src="{{ site.baseurl }}/images/primer-planedefinition.svg" width="45%" float="right">  

A plane definition is an array of one point and three vectors, the point marks the origin of the plane and the vectors represent the three axes. There are some rules to plane definitions, I.e. not every combination of points and vectors is a valid plane. If you create a plane using one of the RhinoScript plane methods you don't have to worry about this, since all the bookkeeping will be done for you. The rules are as follows:

- The axis vectors must be unitized (have a length of 1.0).
- All axis vectors must be perpendicular to each other.
- The x and y axis are ordered anti-clockwise.

The illustration shows how rules #2 and #3 work in practice.

```python
ptOrigin = rs.GetPoint("Plane origin")

ptX = rs.GetPoint("Plane X-axis", ptOrigin)

ptY = rs.GetPoint("Plane Y-axis", ptOrigin)

dX = rs.Distance(ptOrigin, ptX)
dY = rs.Distance(ptOrigin, ptY)
arrPlane = rs.PlaneFromPoints(ptOrigin, ptX, ptY)

rs.AddPlaneSurface(arrPlane, 1.0, 1.0)
rs.AddPlaneSurface(arrPlane, dX, dY)
```

<table>
<tr>
<td>
You will notice that all RhinoScript methods that require plane definitions make sure these demands are met, no matter how poorly you defined the input. 
<br><br>
The adjacent illustration shows how the rs.AddPlaneSurface() call on line 11 results in the red plane, while the rs.AddPlaneSurface() call on line 12 creates the yellow surface which has dimensions equal to the distance between the picked origin and axis points.
</td>
<td width="40%"><img src="{{ site.baseurl }}/images/primer-planecreation.svg" width="100%" height="300" float="right"></td>
</tr>
</table>

We'll only pause briefly at plane definitions since planes, like vectors, are usually only constructive elements. In examples to come they will be used extensively so don't worry about getting the hours in. A more interesting script which uses the *rs.AddPlaneSurface()* method is the one below which populates a surface with so-called surface frames:

```primer
idSurface = rs.GetObject("Surface to frame", 8, True, True)
	
intCount = rs.GetInteger("Number of iterations per direction", 20, 2)

uDomain = rs.SurfaceDomain(idSurface, 0)
vDomain = rs.SurfaceDomain(idSurface, 1)
uStep = (uDomain[1] - uDomain[0]) / intCount
vStep = (vDomain[1] - vDomain[0]) / intCount
	
rs.EnableRedraw(False)
for u in range(uDomain[0],uDomain[1], uStep):
    For v in range(vdomain[0],vDomain[1],vStep):
        pt = rs.EvaluateSurface(idSurface, [u, v]) 
        if rs.Distance(pt, rs.BrepClosestPoint(idSurface, pt)[0]) < 0.1:
            srfFrame = rs.SurfaceFrame(idSurface, [u, v])
            rs.AddPlaneSurface(srfFrame, 1.0, 1.0) 

rs.EnableRedraw(True)
```
Frames are planes which are used to indicate geometrical directions. Both curves, surfaces and textured meshes have frames which identify tangency and curvature in the case of curves and [u] and [v] directions in the case of surfaces and meshes. The script above simply iterates over the [u] and [v] directions of any given surface and adds surface frame objects at all uv coordinates it passes.

<table>
<tr>
<td>
On lines 5 and 6 we determine the domain of the surface in u and v directions and we derive the required stepsize from those limits.
<br><br>
Line 11 and 12 form the main structure of the two-dimensional iteration. You can read such nested For loops as "Iterate through all columns and inside every column iterate through all rows".
<br><br>
Line 14 does something interesting which is not apparent in the adjacent illustration. When we are dealing with trimmed surfaces, those two lines prevent the script from adding planes in cut-away areas. By comparing the point on the (untrimmed) surface to it's projection onto the trimmed surface, we know whether or not the [uv] coordinate in question represents an actual point on the trimmed surface.
</td>
<td width="40%"><img src="{{ site.baseurl }}/images/primersurfaceframes.svg" width="100%" height="300" float="right"></td>
</tr>
</table>

The *rs.SurfaceFrame()* method returns a unitized frame whose axes point in the [u] and [v] directions of the surface. Note that the [u] and [v] directions are not necessarily perpendicular to each other, but we only add valid planes whose x and y axis are always at 90º, thus we ignore the direction of the v-component.

## 8.6 Circles, Ellipses and Arcs

Although the user is never confronted with parametric objects in Rhino, the openNURBS™ kernel has a certain set of mathematical primitives which are stored parametrically. Examples of these are cylinders, spheres, circles, revolutions and sum-surfaces. To highlight the difference between explicit (parametric) and implicit circles:

<img src="{{ site.baseurl }}/images/primer-circleschart.svg" width="100%" float="right">

When adding circles to Rhino through scripting, we can either use the Plane+Radius approach or we can use a 3-Point approach (which is internally translated into Plane+Radius). You may remember that circles are tightly linked with sines and cosines; those lovable, undulating waves. We're going to create a script which packs circles with a predefined radius onto a sphere with another predefined radius. Now, before we start and I give away the answer, I'd like you to take a minute and think about this problem.

The most obvious solution is to start stacking circles in horizontal bands and simply to ignore any vertical nesting which might take place. If you reached a similar solution and you want to keep feeling good about yourself I recommend you skip the following two sentences. This very solution has been found over and over again but for some reason Dave Rusin is usually given as the inventor. Even though Rusin's algorithm isn't exactly rocket science, it is worth discussing the mathematics in advance to prevent -or at least reduce- any confusion when I finally confront you with the code.

Rusin's algorithm works as follows:

- Solve how many circles you can evenly stack from north pole to south pole on the sphere.
- For each of those bands, solve how many circles you can stack evenly around the sphere.
- Do it.

No wait, back up. The first thing to realize is how a sphere actually works. Only once we master spheres can we start packing them with circles. In Rhino, a sphere is a surface of revolution, which has two singularities and a single seam:

<img src="{{ site.baseurl }}/images/primer-uv-map.svg" width="80%" float="right">

The north pole (the black dot in the left most image) and the south pole (the white dot in the same image) are both on the main axis of the sphere and the seam (the thick edge) connects the two. In essence, a sphere is a rectangular plane bent in two directions, where the left and right side meet up to form the seam and the top and bottom edge are compressed into a single point each (a singularity). This coordinate system should be familiar since we use the same one for our own planet. However, our planet is divided into latitude and longitude degrees, whereas spheres are defined by latitude and longitude radians. The numeric domain of the latitude of the sphere starts in the south pole with -½π, reaches 0.0 at the equator and finally terminates with ½π at the north pole. The longitudinal domain starts and stops at the seam and travels around the sphere from 0.0 to 2π. Now you also know why it is called a 'seam' in the first place; it's where the domain suddenly jumps from one value to another, distant one.

We cannot pack circles in the same way as we pack squares in the image above since that would deform them heavily near the poles, as indeed the squares are deformed. We want our circles to remain perfectly circular which means we have to fight the converging nature of the sphere

<table>
<tr>
<td width="30%"><img src="{{ site.baseurl }}/images/primer-sphereuv.svg" width="100%" float="right"></td>
<td>
Assuming the radius of the circles we are about to stack is sufficiently smaller than the radius of the sphere, we can at least place two circles without thinking; one on the north- and one on the south pole. The additional benefit is that these two circles now handsomely cover up the singularities so we are only left with the annoying seam. The next order of business then, is to determine how many circles we need in order to cover up the seam in a straightforward fashion. The length of the seam is half of the circumference of the sphere (see yellow arrow in adjacent illustration).  
<br><br>
Home stretch time, we've collected all the information we need in order to populate this sphere. The last step of the algorithm is to stack circles around the sphere, starting at every seam-circle. We need to calculate the circumference of the sphere at that particular latitude, divide that number by the diameter of the circles and once again find the largest integer value which is smaller than or equal to that result. The equivalent mathematical notation for this is:
<br><br>

$$N_{count} = \left[\frac{2 \cdot R_{sphere} \cdot \cos{\phi}}{2 \cdot R_{circle}} \right]$$  

in case you need to impress anyone…
</td>
</tr>
</table>

```python
def DistributeCirclesOnSphere():
    sphere_radius = rs.GetReal("Radius of sphere", 10.0, 0.01)
    if not sphere_radius: return

    circle_radius = rs.GetReal("Radius of circles", 0.05*sphere_radius, 0.001, 0.5*sphere_radius)
    if not circle_radius: return

    vertical_count = int( (math.pi*sphere_radius)/(2*circle_radius) )

    rs.EnableRedraw(False)
    phi = -0.5*math.pi
    phi_step = math.pi/vertical_count
    while phi<0.5*math.pi:
        horizontal_count = int( (2*math.pi*math.cos(phi)*sphere_radius)/(2*circle_radius) )
        if horizontal_count==0: horizontal_count=1
        theta = 0
        theta_step = 2*math.pi/horizontal_count
        while theta<2*math.pi-1e-8:
            circle_center = (sphere_radius*math.cos(theta)*math.cos(phi), 
                sphere_radius*math.sin(theta)*math.cos(phi), sphere_radius*math.sin(phi))
            circle_normal = rs.PointSubtract(circle_center, (0,0,0))
            circle_plane = rs.PlaneFromNormal(circle_center, circle_normal)
            rs.AddCircle(circle_plane, circle_radius)
            theta += theta_step
        phi += phi_step
    rs.EnableRedraw(True)
```
<img src="{{ site.baseurl }}/images/primer-spherepack.svg" width="45%" float="right">

<table rules="rows">
<tr>
<th>Line</th>	
<th>Description</th>
</tr>
<tr>
<td style="vertical-align:top;text-align:right;padding:0px 10px;">1...6</td>
<td>Collect all custom variables and make sure they make sense. We don't want spheres smaller than 0.01 units and we don't want circle radii larger than half the sphere radius.</td>
</tr>
<tr>
<td style="vertical-align:top;text-align:right;padding:0px 10px;">8</td>
<td>Compute the number of circles from pole to pole. The <i>int()</i> function in VBScript takes a double and returns only the integer part of that number. Hence it always rounds downwards.</td>
</tr>
<tr>
<td style="vertical-align:top;text-align:right;padding:0px 10px;">11...12<br>16...17</td>
<td>phi and theta (Φ and Θ) are typically used to denote angles in spherical space and it's not hard to see why. I could have called them latitude and longitude respectively as well.</td>
</tr>
<tr>
<td style="vertical-align:top;text-align:right;padding:0px 10px;">13</td>
<td>The phi loop runs from -½π to ½π and we need to run it <i>VerticalCount</i> times.</td>
</tr>
<tr>
<td style="vertical-align:top;text-align:right;padding:0px 10px;">14</td>
<td>This is where we calculate how many circles we can fit around the sphere on the current latitude. The math is the same as before, except we also need to calculate the length of the path around the sphere: 2π·R·Cos(Φ)</td>
</tr>
<tr>
<td style="vertical-align:top;text-align:right;padding:0px 10px;">15</td>
<td>If it turns out that we can fit no circles at all at a certain latitude, we're going to get into trouble since we use the HorizontalCount variable as a denominator in the stepsize calculation on line 24. And even my mother knows you cannot divide by zero. However, we know we can always fit at least one circle.</td>
</tr>
<tr>
<td style="vertical-align:top;text-align:right;padding:0px 10px;">18</td>
<td>This loop is essentially the same as the one on line 20, except it uses a different stepsize and a different numeric range ({0.0 <= theta < 2π} instead of {-½π <= phi <= +½π}). The more observant among you will have noticed that the domain of theta reaches from nought up to but not including two pi. If <i>theta</i> would go all the way up to 2π then there would be a duplicate circle on the seam. The best way of preventing a loop to reach a certain value is to subtract a fraction of the stepsize from that value, in this case I have simply subtracted a ludicrously small number (1e-8 = 0.00000001).</td>
</tr>
<tr>
<td style="vertical-align:top;text-align:right;padding:0px 10px;">19...21</td>
<td><i>circle_center</i> will be used to store the center point of the circles we're going to add.  
<i>circle_normal</i> will be used to store the normal of the plane in which these circles reside.  
<i>circle_plane</i> will be used to store the resulting plane definition.</td>
</tr>
<tr>
<td style="vertical-align:top;text-align:right;padding:0px 10px;">19</td>
<td>This is mathematically the most demanding line, and I'm not going to provide a full proof of why and how it works. This is the standard way of translating the spherical coordinates Φ and Θ into Cartesian coordinates x, y and z. 
<br><br>
Further information can be found on [MathWorld.com](http://mathworld.wolfram.com/)</td>
</tr>
<tr>
<td style="vertical-align:top;text-align:right;padding:0px 10px;">20</td>
<td>Once we found the point on the sphere which corresponds to the current values of phi and theta, it's a piece of proverbial cake to find the normal of the sphere at that location. The normal of a sphere at any point on its surface is the inverted vector from that point to the center of the sphere. And that's what we do on line 29, we subtract the sphere origin (always (0,0,0) in this script) from the newly found {x,y,z} coordinate.</td>
</tr>
<tr>
<td style="vertical-align:top;text-align:right;padding:0px 10px;">21...22</td>
<td>We can construct a plane definition from a single point on that plane and a normal vector and we can construct a circle from a plane definition and a radius value. Voila.</td>
</tr>
</table>

### 8.6.1 Ellipses

Ellipses essentially work the same as circles, with the difference that you have to supply two radii instead of just one. Because ellipses only have two mirror symmetry planes and circles possess rotational symmetry (I.e. an infinite number of mirror symmetry planes), it actually does matter a great deal how the base-plane is oriented in the case of ellipses. A plane specified merely by origin and normal vector is free to rotate around that vector without breaking any of the initial constraints.

The following example script demonstrates very clearly how the orientation of the base plane and the ellipse correspond. Consider the standard curvature analysis graph as shown on the left:

<img src="{{ site.baseurl }}/images/primer-curvaturespline.svg" width="80%" float="right">

It gives a clear impression of the range of different curvatures in the spline, but it doesn't communicate the helical twisting of the curvature very well. Parts of the spline that are near-linear tend to have a garbled curvature since they are the transition from one well defined bend to another. The arrows in the left image indicate these areas of twisting but it is hard to deduce this from the curvature graph alone. The upcoming script will use the curvature information to loft a surface through a set of ellipses which have been oriented into the curvature plane of the local spline geometry. The ellipses have a small radius in the bending plane of the curve and a large one perpendicular to the bending plane. Since we will not be using the strength of the curvature but only its orientation, small details will become very apparent. 

```python
def FlatWorm():
    curve_object = rs.GetObject("Pick a backbone curve", 4, True, False)
    if not curve_object: return

    samples = rs.GetInteger("Number of cross sections", 100, 5)
    if not samples: return

    bend_radius = rs.GetReal("Bend plane radius", 0.5, 0.001)
    if not bend_radius: return

    perp_radius = rs.GetReal("Ribbon plane radius", 2.0, 0.001)
    if not perp_radius: return

    crvdomain = rs.CurveDomain(curve_object)

    crosssections = []
    t_step = (crvdomain[1]-crvdomain[0])/samples
    t = crvdomain[0]
    for t in rs.frange(crvdomain[0], crvdomain[1], t_step):
        crvcurvature = rs.CurveCurvature(curve_object, t)
        crosssectionplane = None
        if not crvcurvature:
            crvPoint = rs.EvaluateCurve(curve_object, t)
            crvTangent = rs.CurveTangent(curve_object, t)
            crvPerp = (0,0,1)
            crvNormal = rs.VectorCrossProduct(crvTangent, crvPerp)
            crosssectionplane = rs.PlaneFromFrame(crvPoint, crvPerp, crvNormal)
        else:
            crvPoint = crvcurvature[0]
            crvTangent = crvcurvature[1]
            crvPerp = rs.VectorUnitize(crvcurvature[4])
            crvNormal = rs.VectorCrossProduct(crvTangent, crvPerp)
            crosssectionplane = rs.PlaneFromFrame(crvPoint, crvPerp, crvNormal)

        if crosssectionplane:
            csec = rs.AddEllipse(crosssectionplane, bend_radius, perp_radius)
            crosssections.append(csec)
        t += t_step

    if not crosssections: return
    rs.AddLoftSrf(crosssections)
    rs.DeleteObjects(crosssections)
```


<table rules="rows">
<tr>
<th>Line</th>	
<th>Description</th>
</tr>
<tr>
<td style="vertical-align:top;text-align:right;padding:0px 10px;">16</td>
<td><i>crosssections</i> is a list where we will store all our ellipse IDs. We need to remember all the ellipses we add since they have to be fed to the <i>rs.AddLoftSrf()</i> method. <i>crosssectionplane</i> will contain the base plane data for every individual ellipse, we do not need to remember these planes so we can afford to overwrite the old value with any new one.
You'll notice I'm violating a lot of naming conventions from paragraph [2.3.5 Using Variables]. If you want to make something of it we can take it outside. </td>
</tr>
<tr>
<td style="vertical-align:top;text-align:right;padding:0px 10px;">19</td>
<td>We'll be walking along the curve with equal parameter steps. This is arguably not the best way, since we might be dealing with a polycurve which has wildly different parameterizations among its subcurves. This is only an example script though so I wanted to keep the code to a minimum. We're using the same trick as before in the header of the loop to ensure that the final value in the domain is included in the calculation. By extending the range of the loop by one billionth of a parameter we circumvent the 'double noise problem' which might result from multiple additions of doubles.</td>
</tr>
<tr>
<td style="vertical-align:top;text-align:right;padding:0px 10px;">20</td>
<td>The <i>rs.CurveCurvature()</i> method returns a whole set of data to do with curvature analysis. However, it will fail on any linear segment (the radius of curvature is infinite on linear segments).</td>
</tr>
<tr>
<td style="vertical-align:top;text-align:right;padding:0px 10px;">22...27</td>
<td>Hence, if it fails we have to collect the standard information in the old fashioned way. We also have to pick a <i>crvPerp</i> vector since none is available. We could perhaps use the last known one, or look at the local plane of the curve beyond the current -unsolvable- segment, but I've chosen to simply use a z-axis vector by default.</td>
</tr>
<tr>
<td style="vertical-align:top;text-align:right;padding:0px 10px;">28...32</td>
<td>If the curve does have curvature at t, then we extract the required information directly from the curvature data.</td>
</tr>
<tr>
<td style="vertical-align:top;text-align:right;padding:0px 10px;">33</td>
<td>Construct the plane for the ellipse.</td>
</tr>
<tr>
<td style="vertical-align:top;text-align:right;padding:0px 10px;">35...37</td>
<td>Add the ellipse to the file and append the new ellipse curve ID csec to the list <i>crosssections</i>.</td>
</tr>
<tr>
<td style="vertical-align:top;text-align:right;padding:0px 10px;">40...42</td>
<td>Create a lofted surface through all ellipses and delete the curves afterwards.</td>
</tr>
</table>

### 8.6.2 Arcs

Since the topic of Arcs isn't much different from the topic of Circles, I thought it would be a nice idea to drag in something extra. This something extra is what we programmers call "recursion" and it is without doubt the most exciting thing in our lives (we don't get out much). Recursion is the process of self-repetition. Like loops which are iterative and execute the same code over and over again, recursive functions call themselves and thus also execute the same code over and over again, but this process is hierarchical. It actually sounds harder than it is. One of the success stories of recursive functions is their implementation in binary trees which are the foundation for many search and classification algorithms in the world today. I'll allow myself a small detour on the subject of recursion because I would very much like you to appreciate the power that flows from the simplicity of the technique. Recursion is unfortunately one of those things which only become horribly obvious once you understand how it works. 

Imagine a box in 3D space which contains a number of points within its volume. This box exhibits a single behavioral pattern which is recursive. The recursive function evaluates a single conditional statement: {when the number of contained points exceeds a certain threshold value then subdivide into 8 smaller boxes, otherwise add yourself to the document}. It would be hard to come up with an easier If…Else statement. Yet, because this behavior is also exhibited by all newly created boxes, it bursts into a chain of recursion, resulting in the voxel spaces in the images below:

<img src="{{ site.baseurl }}/images/primer-threshold.svg" width="80%" float="right">

The input in these cases was a large pointcloud shaped like the upper half of a sphere. There was also a dense spot with a higher than average concentration of points. Because of the approximating pattern of the subdivision, the recursive cascade results in these beautiful stacks. Trying to achieve this result without the use of recursion would entail a humongous amount of bookkeeping and many, many lines of code.
Before we can get to the cool bit we have to write some of the supporting functions, which -I hate to say it- once again involve goniometry (the mathematics of angles).

The problem: adding an arc using the start point, end point and start direction. As you will be aware there is a way to do this directly in Rhino using the mouse. In fact a brief inspection yields 14 different ways in which arcs can be drawn in Rhino manually and yet there are only two ways to add arcs through scripting:

- rs.AddArc(Plane, Radius, Angle)
- rs.AddArc3Pt(Point, Point, Point)

The first way is very similar to adding circles using plane and radius values, with the added argument for sweep angle. The second way is also similar to adding circles using a 3-point system, with the difference that the arc terminates at the first and second point. There is no direct way to add arcs from point A to point B while constrained to a start tangent vector. We're going to have to write a function which translates the desired Start-End-Direction approach into a 3-Point approach. Before we tackle the math, let's review how it works:

<table>
<tr>
<td width="30%"><img src="{{ site.baseurl }}/images/primer-arcs.svg" width="100%" float="right"></td>
<td>
We start with two points {A} & {B} and a vector definition {D}. The arc we're after is the red curve, but at this point we don't know how to get there yet. Note that this problem might not have a solution if {D} is parallel or anti-parallel to the line from {A} to {B}. If you try to draw an arc like that in Rhino it will not work. Thus, we need to add some code to our function that aborts when we're confronted with unsolvable input.  
<br><br>
We're going to find the coordinates of the point in the middle of the desired arc {M}, so we can use the 3Point approach with {A}, {B} and {M}. As the illustration on the left indicates, the point in the middle of the arc is also on the line perpendicular from the middle {C} of the baseline.
<br><br>
The halfway point on the arc also happens to lie on the bisector between {D} and the baseline vector. We can easily construct the bisector of two vectors in 3D space by process of unitizing and adding both vectors. In the illustration on the left the bisector is already pointing in the right direction, but it still hasn't got the correct length.
<br><br> 
We can compute the correct length using the standard "Sin-Cos-Tan right triangle rules": 

The triangle we have to solve has a 90º angle in the lower right corner, a is the angle between the baseline and the bisector, the length of the bottom edge of the triangle is half the distance between {A} and {B} and we need to compute the length of the slant edge (between {A} and {M}).
</td>
</tr>
</table>

The relationship between a and the lengths of the sides of the triangle is: 

$$\cos({\alpha})=\frac{0.5D}{?} \gg \frac{1}{\cos({\alpha})}=\frac{?}{0.5D} \gg \frac{0.5D}{\cos({\alpha})} = ?$$  

We now have the equation we need in order to solve the length of the slant edge. The only remaining problem is cos(a). In the paragraph on vector mathematics (6.2 Points and Vectors) the vector dotproduct is briefly introduced as a way to compute the angle between two vectors. When we use unitized vectors, the arccosine of the dotproduct gives us the angle between them. This means the dotproduct returns the cosine of the angle between these vectors. This is a very fortunate turn of events since the cosine of the angle is exactly the thing we're looking for. In other words, the dotproduct saves us from having to use the cosine and arccosine functions altogether. Thus, the distance between {A} and {M} is the result of:


```python
(0.5 * rs.Distance(A, B)) / rs.VectorDotProduct(D, Bisector)
```

```python
def AddArcDir(ptStart, ptEnd, vecDir):
    vecBase = rs.PointSubtract(ptEnd, ptStart)
    if rs.VectorLength(vecBase)==0.0: return
    
    if rs.IsVectorParallelTo(vecBase, vecDir): return

    vecBase = rs.VectorUnitize(vecBase)
    vecDir = rs.VectorUnitize(vecDir)

    vecBisector = rs.VectorAdd(vecDir, vecBase)
    vecBisector = rs.VectorUnitize(vecBisector)
    
    dotProd = rs.VectorDotProduct(vecBisector, vecDir)
    midLength = (0.5*rs.Distance(ptStart, ptEnd))/dotProd
    
    vecBisector = rs.VectorScale(vecBisector, midLength)
    return rs.AddArc3Pt(ptStart, rs.PointAdd(ptStart, vecBisector), ptEnd)
```

<table rules="rows">
<tr>
<th>Line</th>	
<th>Description</th>
</tr>
<tr>
<td style="vertical-align:top;text-align:right;padding:0px 10px;">1</td>
<td>The <i>ptStart</i> argument indicates the start of the arc, <i>ptEnd</i> the end and <i>vecDir</i> the direction at <i>ptStart</i>. This function will behave just like the <i>rs.AddArc3Pt()</i> method. It takes a set of arguments and returns the identifier of the created curve object if successful. If no curve was added the function does not return anything - that is, the resulting assignment will be <i>None</i>.</td>
</tr>
<tr>
<td style="vertical-align:top;text-align:right;padding:0px 10px;">2</td>
<td>Create the baseline vector (from {A} to {B}), by subtracting {A} from {B}.</td>
</tr>
<tr>
<td style="vertical-align:top;text-align:right;padding:0px 10px;">3</td>
<td>If {A} and {B} are coincident, then the subtraction from line 2 will result in a vector with a length of 0 and no solution is possible. Actually, there is an infinite number of solutions so we wouldn't know which one to pick.</td>
</tr>
<tr>
<td style="vertical-align:top;text-align:right;padding:0px 10px;">5</td>
<td>If vecDir is parallel (or anti-parallel) to the baseline vector, then no solution is possible at all.</td>
</tr>
<tr>
<td style="vertical-align:top;text-align:right;padding:0px 10px;">7...8</td>
<td>Make sure all vector definitions so far are unitized - that is, they all have a vector length value of one</td>
</tr>
<tr>
<td style="vertical-align:top;text-align:right;padding:0px 10px;">10...11</td>
<td>Create the bisector vector and unitize it.</td>
</tr>
<tr>
<td style="vertical-align:top;text-align:right;padding:0px 10px;">13</td>
<td>Compute the dotproduct between the bisector and the direction vector. Since the bisector is exactly halfway between the direction vector and baseline vector (indeed, that is the point to its existence), we could just as well have calculated the dotproduct between it and the baseline vector.</td>
</tr>
<tr>
<td style="vertical-align:top;text-align:right;padding:0px 10px;">14</td>
<td>Compute the distance between ptStart and the center point of the desired arc.</td>
</tr>
<tr>
<td style="vertical-align:top;text-align:right;padding:0px 10px;">16</td>
<td>Resize the (unitized) bisector vector to match this length.</td>
</tr>
<tr>
<td style="vertical-align:top;text-align:right;padding:0px 10px;">17</td>
<td>Create an arc using the start, end and midpoint arguments, return the ID.</td>
</tr>
</table>


<table>
<tr>
<td>
We need this function in order to build a recursive tree-generator which outputs trees made of arcs. Our trees will be governed by a set of five variables but -due to the 
flexible nature of the recursive paradigm- it will be very easy to add more behavioral patterns. The growing algorithm as implemented in this example is very simple and doesn't allow a great deal of variation.
<br><br>
The five base parameters are:
<br><br>
Propagation factor
<ol>
<li>Twig length</li>
<li>Twig length mutation</li>
<li>Twig angle</li>
<li>Twig angle mutation</li>
</ol>
</td>
<td width="50%"><img src="{{ site.baseurl }}/images/primer-arctree.svg" width="100%" height="300" float="right"></td>
</tr>
</table>

The propagation-factor is a numeric range which indicates the minimum and maximum number of twigs that grow at the end of every branch. This is a totally random affair, which is why it is called a "factor" rather than a "number". More on random numbers in a minute. The twig-length and twig-length-mutation variables control the -as you probably guessed- length of the twigs and how the length changes with every twig generation. The twig-angle and twig-angle-mutation work in a similar fashion.

The actual recursive bit of this algorithm will not concern itself with the addition and shape of the twig-arcs. This is done by a supporting function which we have to write before we can start growing trees. The problem we have when adding new twigs, is that we want them to connect smoothly to their parent branch. We've already got the plumbing in place to make tangency continuous arcs, but we have no mechanism yet for picking the end-point. In our current plant-scheme, twig growth is controlled by two factors; length and angle. However, since more than one twig might be growing at the end of a branch there needs to be a certain amount of random variation to keep all the twigs from looking the same.

<table>
<tr>
<td>
The adjacent illustration shows the algorithm we'll be using for twig propagation. The red curve is the branch-arc and we need to populate the end with any number of twig-arcs. Point {A} and Vector {D} are dictated by the shape of the branch but we are free to pick point {B} at random provided we remain within the limits set by the length and angle constraints. The complete set of possible end-points is drawn as the yellow cone. We're going to use a sequence of Vector methods to get a random point {B} in this shape:

<ol>
<li>Create a new vector {T} parallel to {D}</li>
<li>Resize {T} to have a length between {Lmin} and {Lmax}</li>
<li>Mutate {T} to deviate a bit from {D}</li>
<li>Rotate {T} around {D} to randomize the orientation</li>
</ol>
</td>
<td width="30%"><img src="{{ site.baseurl }}/images/primer-branchpropagation2.svg" width="100%" height="300" float="right"></td>
</tr>
</table>

```python
def RandomPointInCone( origin, direction, minDistance, maxDistance, maxAngle):
    vecTwig = rs.VectorUnitize(direction)
    vecTwig = rs.VectorScale(vecTwig, minDistance + random.random()*(maxDistance-minDistance))
    MutationPlane = rs.PlaneFromNormal((0,0,0), vecTwig)
    vecTwig = rs.VectorRotate(vecTwig, random.random()*maxAngle, MutationPlane[1])
    vecTwig = rs.VectorRotate(vecTwig, random.random()*360, direction)
    return rs.PointAdd(origin, vecTwig)
```

<table rules="rows">
<tr>
<th>Line</th>	
<th>Description</th>
</tr>
<tr>
<td style="vertical-align:top;text-align:right;padding:0px 10px;">1</td>
<td><i>origin</i> is synonymous with point {A}.
<i>direction</i> is synonymous with vector {D}.
<i>minDistance</i> and MaxDistance indicate the length-wise domain of the cone.
<i>maxAngle</i> is a value which specifies the angle of the cone (in degrees, not radians).</td>
</tr>
<tr>
<td style="vertical-align:top;text-align:right;padding:0px 10px;">2...3</td>
<td>Create a new vector parallel to <i>Direction</i> and resize it to be somewhere between <i>MinDistance</i> and <i>MaxDistance</i>. I'm using the <i>random()</i> function here which is a Python pseudo-random-number frontend. It always returns a random value between zero and one.</td>
</tr>
<tr>
<td style="vertical-align:top;text-align:right;padding:0px 10px;">4</td>
<td>In order to mutate <i>vecTwig</i>, we need to find a parallel vector. since we only have one vector here we cannot directly use the <i>Rhino.VectorCrossProduct()</i> method, so we'll construct a plane and use its x-axis. This vector could be pointing anywhere, but always perpendicular to <i>vecTwig</i>.</td>
</tr>
<tr>
<td style="vertical-align:top;text-align:right;padding:0px 10px;">5</td>
<td>Mutate <i>vecTwig</i> by rotating a random amount of degrees around the plane x-axis.</td>
</tr>
<tr>
<td style="vertical-align:top;text-align:right;padding:0px 10px;">6</td>
<td>Mutate <i>vecTwig</i> again by rotating it around the <i>Direction</i> vector. This time the random angle is between 0 and 360 degrees.</td>
</tr>
<tr>
<td style="vertical-align:top;text-align:right;padding:0px 10px;">7</td>
<td>Create the new point as inferred by <i>Origin</i> and <i>vecTwig</i>.</td>
</tr>
</table>

One of the definitions Wikipedia has to offer on the subject of recursion is: "In order to understand recursion, one must first understand recursion." Although this is obviously just meant to be funny, there is an unmistakable truth as well. The upcoming script is recursive in every definition of the word, it is also quite short, it produces visually interesting effects and it is quite clearly a very poor realistic plant generator. The perfect characteristics for exploration by trial-and-error. Probably more than any other example script in this primer this one is a lot of fun to play around with. Modify, alter, change, mangle and bend it as you see fit.

There is a set of rules to which any working recursive function must adhere. It must place at least one call to itself somewhere before the end and must have a way of exiting without placing any calls to itself. If the first condition is not met the function cannot be called recursive and if the second condition is not met it will call itself until time stops (or rather until the call-stack memory in your computer runs dry).

Lo and behold! 
A mere 21 lines of code to describe the growth of an entire tree.

```python
def RecursiveGrowth( ptStart, vecDir, props, generation):
    minTwigCount, maxTwigCount, maxGenerations, maxTwigLength, lengthMutation, maxTwigAngle,...      
        angleMutation = props
    if generation>maxGenerations: return

    #Copy and mutate the growth-properties
    newProps = props
    maxTwigLength *= lengthMutation
    maxTwigAngle *= angleMutation
    if maxTwigAngle>90: maxTwigAngle=90

    #Determine the number of twigs (could be less than zero)
    newprops = minTwigCount, maxTwigCount, maxGenerations, maxTwigLength, lengthMutation,...
        maxTwigAngle, angleMutation
    maxN = int( minTwigCount+random.random()*(maxTwigCount-minTwigCount) )
    for n in range(1,maxN):
        ptGrow = RandomPointInCone(ptStart, vecDir, 0.25*maxTwigLength, maxTwigLength,... 
            maxTwigAngle)
        newTwig = AddArcDir(ptStart, ptGrow, vecDir)
        if newTwig:
            vecGrow = rs.CurveTangent(newTwig, rs.CurveDomain(newTwig)[1])
            RecursiveGrowth(ptGrow, vecGrow, newProps, generation+1)
```            


<table rules="rows">
<tr>
<th>Line</th>	
<th>Description</th>
</tr>
<tr>
<td style="vertical-align:top;text-align:right;padding:0px 10px;">1</td>
<td>A word on the function signature. Apart from the obvious arguments <i>ptStart</i> and <i>vecDir</i>, this function takes an tuple and a generation counter. The tuple contains all our growth variables. Since there are seven of them in total I didn't want to add them all as individual arguments. Also, this way it is easier to add parameters without changing function calls. The generation argument is an integer telling the function which twig generation it is in. Normally a recursive function does not need to know its depth in the grand scheme of things, but in our case we're making an exception since the number of generations is an exit threshold.</td>
</tr>
<tr>
<td style="vertical-align:top;text-align:right;padding:0px 10px;">2</td>
<td>For readability, we will break our tuple into individual variables. On the assignment side, the variables are listed in the order that they appear in the tuple. The properties tuple consists of the following items:

<img src="{{ site.baseurl }}/images/primer-data-table.svg" width="100%" float="right">
</td>
</tr>
<tr>
<td style="vertical-align:top;text-align:right;padding:0px 10px;">3</td>
<td>If the current generation exceeds the generation limit (which is stored at the third element in the properties tuple, and broken out to the variable maxGenerations) this function will abort without calling itself. Hence, it will take a step back on the recursive hierarchy.</td>
</tr>
<tr>
<td style="vertical-align:top;text-align:right;padding:0px 10px;">6</td>
<td>This is where we make a copy of the properties. You see, when we are going to grow new twigs, those twigs will be called with mutated properties, however we require the unmutated properties inside this function instance.</td>
</tr>
<tr>
<td style="vertical-align:top;text-align:right;padding:0px 10px;">7...9</td>
<td>Mutate the copied properties. I.e. multiply the maximum-twig-length by the twig-length-mutation factor and do the same for the angle. We must take additional steps to ensure the angle doesn't go berserk so we're limiting the mutation to within the 90 degree realm.</td>
</tr>
<tr>
<td style="vertical-align:top;text-align:right;padding:0px 10px;">13</td>
<td><i>maxN</i> is an integer which indicated the number of twigs we are about to grow. <i>maxN</i> is randomly picked between the two allowed extremes (<i>Props(0)</i> and <i>Props(1)</i>). The <i>random()</i> function generates a number between zero and one which means that maxN can become any value between and including the limits.</td>
</tr>
<tr>
<td style="vertical-align:top;text-align:right;padding:0px 10px;">15</td>
<td>This is where we pick a point at random using the unmutated properties. The length constraints we're using is hard coded to be between the maximum allowed length and a quarter of the maximum allowed length. There is nothing in the universe which suggests a factor of 0.25, it is purely arbitrary. It does however have a strong effect on the shape of the trees we're growing. It means it is impossible to accurately specify a twig length. There is a lot of room for experimentation and change here.</td>
</tr>
<tr>
<td style="vertical-align:top;text-align:right;padding:0px 10px;">16</td>
<td>We create the arc that belongs to this twig.</td>
</tr>
<tr>
<td style="vertical-align:top;text-align:right;padding:0px 10px;">17</td>
<td>If the distance between <i>ptStart</i> and <i>ptGrow</i> was 0.0 or if <i>vecDir</i> was parallel to <i>ptStart</i> » <i>ptGrow</i> then the arc could not be added. We need to catch this problem in time.</td>
</tr>
<tr>
<td style="vertical-align:top;text-align:right;padding:0px 10px;">18</td>
<td>We need to know the tangent at the end of the newly created arc curve.  The domain of a curve consists of two values (a lower and an upper bound). <i>Rhino.CurveDomain(newTwig)(1)</i> will return the upper bound of the domain. This is the same as calling:

<code lang="python">
crvDomain = rs.CurveDomain(newTwig)
vecGrow = rs.CurveTangent(newTwig, crvDomain[1])
</code>

</td>
</tr>
<tr>
<td style="vertical-align:top;text-align:right;padding:0px 10px;">19</td>
<td>Awooga! Awooga! A function calling itself! This is it! We made it!
The thing to realize is that the call is now different. We're putting in different arguments which means this new function instance behaves differently than the current function instance.</td>
</tr>
</table>

It would have been possible to code this tree-generator in an iterative (For loops) fashion. The tree would look the same even though the code would be very different (probably a lot more lines). The order in which the branches are added would very probably also have differed. The trees below are archetypal, digital trees, the one on the left generated using iteration, the one on the right generated using recursion. Note the difference in branch order. If you look carefully at the recursive function on the previous page you'll probably be able to work out where this difference comes from...

<img src="{{ site.baseurl }}/images/primer_iterativetree_vs_recursivetree.svg" width="80%" float="right">

A small comparison table for different setting combinations. Please note that the trees have a very high random component.

<img src="{{ site.baseurl }}/images/primer-treechart.svg" width="100%" float="right">

## 8.7 Nurbs-curves

Circles and arcs are all fine and dandy, but they cannot be used to draw freeform shapes. For that you need splines. The worlds most famous spline is probably the Bézier curve, which was developed in 1962 by the French engineer *Pierre Bézier* while he was working for Renault. Most splines used in computer graphics these days are variations on the Bézier spline, and they are thus a surprisingly recent arrival on the mathematical scene. Other ground-breaking work on splines was done by *Paul de Casteljau* at Citroën and *Carl de Boor* at General Motors. The thing that jumps out here is the fact that all these people worked for car manufacturers. With the increase in engine power and road quality, the automobile industry started to face new problems halfway through the twentieth century, one of which was aerodynamics. New methods were needed to design mass-production cars that had smooth, fluent curves as opposed to the tangency and curvature fractured shapes of old. They needed mathematically accurate, freely adjustable geometry. Enter splines.

Before we start with NURBS curves (the mathematics of which are a bit too complex for a scripting primer) I'd like to give you a sense of how splines work in general and Béziers work in particular. I'll explain the de Casteljau algorithm which is a very straightforward way of evaluating properties of simple splines. In practice, this algorithm will rarely be used since its performance is worse than alternate approaches, but due to its visual appeal it is easier to 'get a feel' for it.

<img src="{{ site.baseurl }}/images/primer-nurbsalgorithm.svg" width="100%" float="right">

Splines limited to four control points were not the end of the revolution of course. Soon, more advanced spline definitions were formulated one of which is the NURBS curve. (Just to set the record straight; NURBS stands for Non-Uniform Rational [Basic/Basis] Spline and not Bézier-Spline as some people think. In fact, the Rhino help file gets it right, but I doubt many of you have read the glossary section, I only found out just now.) Bézier splines are a subset of NURBS curves, meaning that every Bézier spline can be represented by a NURBS curve, but not the other way around. Other curve types still in use today (but not available in Rhino) are Hermite, Cardinal, Catmull-Rom, Beta and Akima splines, but this is not a complete list. Hermite curves for example are used by the Bongo animation plug-in to smoothly transform objects through a number of keyframes.

In addition to control point locations, NURBS curves have additional properties such as the degree, knot-vectors and weights. I'm going to assume that you already know how weight factors work (if you don't, it's in the Rhino help file under [NURBS About]) so I won't discuss them here. Instead, we'll continue with the correlation between degrees and knot-vectors. 

Every NURBS curve has a number associated with it which represents the degree. The degree of a curve is always a positive integer between and including 1 and 11. The degree of a curve is written as DN. Thus D1 is a degree one curve and D3 is a degree three curve. The table on the next page shows a number of curves with the exact same control-polygon but with different degrees. In short, the degree of a curve determines the range of influence of control points. The higher the degree, the larger the range.

As you will recall from the beginning of this section, a quadratic Bézier curve is defined by four control points. A quadratic NURBS curve however can be defined by any number of control points (any number larger than three that is), which in turn means that the entire curve consists of a number of connected pieces. The illustration below shows a D3 curve with 10 control points. All the individual pieces have been given a different color. As you can see each piece has a rather simple shape; a shape you could approximate with a traditional, four-point Bézier curve. Now you know why NURBS curves and other splines are often described as "piece-wise curves".

<img src="{{ site.baseurl }}/images/primer-piecewisecurve.svg" width="100%" float="right">

The shape of the red piece is entirely dictated by the first four control points. In fact, since this is a D3 curve, every piece is defined by four control points. So the second (orange) piece is defined by points {A; B; C; D}. The big difference between these pieces and a traditional Bézier curve is that the pieces stop short of the local control polygon. Instead of going all the way to {D}, the orange piece terminates somewhere in the vicinity of {C} and gives way to the green piece. Due to the mathematical magic of spline curves, the orange and green pieces fit perfectly, they have an identical position, tangency and curvature at point 4.

As you may or may not have guessed at this point, the little circles between pieces represent the knot-vector of this curve. This D3 curve has ten control points and twelve knots (0~11). This is not a coincidence, the number of knots follows directly from the number of points and the degree:

$$K_N = P_N + (D-1)$$

Where $${K_N}$$ is the knot count, $${P_N}$$ is the point count and $${D}$$ is the degree.

In the image on the previous page, the red and purple pieces do in fact touch the control polygon at the beginning and end, but we have to make some effort to stretch them this far. This effort is called "clamping", and it is achieved by stacking a lot of knots together. You can see that the number of knots we need to collapse in order to get the curve to touch a control-point is the same as the degree of the curve:

<img src="{{ site.baseurl }}/images/primer-curveknot.svg" width="100%" float="right">

A clamped curve always has a bunch of knots at the beginning and end (periodic curves do not, but we'll get to that later). If a curve has knot clusters on the interior as well, then it will touch one of the interior control points and we have a kinked curve. There is a lot more to know about knots, but I suggest we continue with some simple nurbs curves and let Rhino worry about the knot vector for the time being.

### 8.7.1 Control-point curves

<table>
<tr>
<td>
The _FilletCorners command in Rhino puts filleting arcs across all sharp kinks in a polycurve. Since fillet curves are tangent arcs, the corners have to be planar. All flat curves though can always be filleted as the image to the right shows.
<br><br>
The input curve {A} has nine G0 corners (filled circles) which qualify for a filleting operation and three G1 corners (empty circles) which do not. Since each segment of the polycurve has a length larger than twice the fillet radius, none of the fillets overlap and the result is a predictable curve {B}.
<br><br>
Since blend curves are freeform they are allowed to twist and curl as much as they please. They have no problem with non-planar segments. Our assignment for today is to make a script which inserts blend corners into polylines. We're not going to handle polycurves (with freeform curved segments) since that would involve quite a lot of math and logic which goes beyond this simple curve introduction. This unfortunately means we won't actually be making non-planar blend corners. 
</td>
<td width="30%"><img src="{{ site.baseurl }}/images/primer-filletcorners.svg" width="100%" height="300" float="right"></td>
</tr>
</table>

The logic of our BlendCorners script is simple:

- Iterate though all segments of the polyline.
- From the beginning of the segment $${A}$$, place an extra control point $${W_1}$$ at distance $${R}$$.
- From the end of the segment $${B}$$, place an extra control point $${W_2}$$ at distance $${R}$$.
- Put extra control-points halfway between $${A; W_1; W_2; B}$$.
- Insert a $$D^5$$ nurbs curve using those new control points.

Or, in graphic form:

<img src="{{ site.baseurl }}/images/primer-blendcurved5.svg" width="100%" float="right">

The first image shows our input curve positioned on a unit grid. The shortest segment has a length of 1.0, the longest segment a length of 6.0. If we're going to blend all corners with a radius of 0.75 (the circles in the second image) we can see that one of the edges has a conflict of overlapping blend radii.

The third image shows the original control points (the filled circles) and all blend radius control points (the empty circles), positioned along every segment with a distance of {R} from its nearest neighbor The two red control points have been positioned 0.5 units away (half the segment length) from their respective neighbours.

Finally, the last image shows all the control points that will be added in between the existing control points. Once we have an ordered array of all control points (ordered as they appear along the original polyline) we can create a $$D^5$$ curve using *rs.AddCurve()*.

```python
def blendcorners():
    polyline_id = rs.GetObject("Polyline to blend", 4, True, True)
    if not polyline_id: return
    vertices = rs.PolylineVertices(polyline_id)
    if not vertices: return
    radius = rs.GetReal("Blend radius", 1.0, 0.0)
    if radius is None: return

    between = lambda a,b: (a+b)/2.0
    newverts = []
    for i in range(len(vertices)-1):
        a = vertices[i]
        b = vertices[i+1]
        segmentlength = rs.Distance(a, b)
        vec_segment = rs.PointSubtract(b, a)
        vec_segment = rs.VectorUnitize(vec_segment)

        if radius<(0.5*segmentlength):
            vec_segment = rs.VectorScale(vec_segment, radius)
        else:
            vec_segment = rs.VectorScale(vec_segment, 0.5*segment_length)

        w1 = rs.VectorAdd(a, vec_segment)
        w2 = rs.VectorSubtract(b, vec_segment)
        newverts.append(a)
        newverts.append(between(a,w1))
        newverts.append(w1)
        newverts.append(between(w1,w2))
        newverts.append(w2)
        newverts.append(between(w2,b))
    newverts.append(vertices[len(vertices)-1])
    rs.AddCurve(newverts, 5)
    rs.DeleteObject(polyline_id)
```

<table rules="rows">
<tr>
<th>Line</th>	
<th>Description</th>
</tr>
<tr>
<td style="vertical-align:top;text-align:right;padding:0px 10px;">2...7</td>
<td>These calls prompt the user for a polyline, get the polyline's vertices, and then promps the user for the radius of blending. Since David Rutten is the only one allowed to be so careless as to not check for None values (and we are not David Rutten) each of these operations is followed by a failure check.</td>
</tr>
<tr>
<td style="vertical-align:top;text-align:right;padding:0px 10px;">9...10</td>
<td>Sometimes, an operation that is needed within a loop results in the same values in each loop iteration. In cases like this, programs can be made much more efficient, by performing these operations before entering the loop. The variables between and <i>newverts</i> will not be used until line 25, but obtaining them here at lines 9 and 10 will make the script much more efficient.
</td>
</tr>
<tr>
<td style="vertical-align:top;text-align:right;padding:0px 10px;">11</td>
<td>Begin a loop for each segment in the polyline. </td>
</tr>
<tr>
<td style="vertical-align:top;text-align:right;padding:0px 10px;">12...13</td>
<td>Store <i>A</i> and <i>B</i> coordinates for easy reference.</td>
</tr>
<tr>
<td style="vertical-align:top;text-align:right;padding:0px 10px;">15...21</td>
<td><i>vec_segment</i> is a scaled vector that points from <i>A</i> to <i>B</i> with a length of <i>radius</i>.
Calculate the <i>vec_segment</i> vector. Typically this vector has length <i>radius</i>, but if the current polyline segment is too short to contain two complete radii, then adjust the <i>vec_segment</i> accordingly.</td>
</tr>
<tr>
<td style="vertical-align:top;text-align:right;padding:0px 10px;">23...24</td>
<td>Calculate <i>W1</i> and <i>W2</i>.</td>
</tr>
<tr>
<td style="vertical-align:top;text-align:right;padding:0px 10px;">25...30</td>
<td>Store all points (except <i>B</i>) in the <i>newverts</i> list.</td>
</tr>
<tr>
<td style="vertical-align:top;text-align:right;padding:0px 10px;">31</td>
<td>Append the last point of the polyline to the <i>newverts</i> list. We've omitted <i>B</i> everywhere because the <i>A</i> of the next segment has the same location and we do not want coincident control-points. The last segment has no next segment, so we need to make sure <i>B</i> is included this time.</td>
</tr>
<tr>
<td style="vertical-align:top;text-align:right;padding:0px 10px;">32...33</td>
<td>Create a new D5 nurbs curve and delete the original.</td>
</tr>
</table>

### 8.7.2 Interpolated curves

When creating control-point curves it is very difficult to make them go through specific coordinates. Even when tweaking control-points this would be an arduous task. This is why commands like *_HBar* are so important. However, if you need a curve to go through many points, you're better off creating it using an interpolated method rather than a control-point method. The *_InterpCrv* and *_InterpCrvOnSrf* commands allow you to create a curve that intersects any number of 3D points and both of these methods have an equivalent in RhinoScript.

To demonstrate, we're going to create a script that creates iso-distance-curves on surfaces rather than the standard iso-parameter-curves, or "isocurves" as they are usually called.  Isocurves, thus connect all the points in surface space that share a similar u or v value. Because the progression of the domain of a surface is not linear (it might be compressed in some places and stretched in others, especially near the edges where the surface has to be clamped), the distance between isocurves is not guaranteed to be identical either.

The description of our algorithm is very straightforward, but I promise you that the actual script itself will be the hardest thing you've ever done.

<img src="{{ site.baseurl }}/images/primer-isocurves.svg" width="100%" float="right">

Our script will take any base surface (image A) and extract a number of isocurves (image B). Then, every isocurve is trimmed to a specific length (image C) and the end-points are connected to give the iso-distance-curve (the red curve in image D). Note that we are using isocurves in the v-direction to calculate the iso-distance-curve in the u-direction. This way, it doesn't matter much that the spacing of isocurves isn't distributed equally. Also note that this method is only useful for offsetting surface edges as opposed to *_OffsetCrvOnSrf* which can offset any curve.

We can use the RhinoScript methods *rs.ExtractIsoCurve()* and *rs.AddInterpCrvOnSrf()* for steps B and D, but step C is going to take some further thought. It is possible to divide the extracted isocurve using a fixed length, which will give us a whole list of points, the second of which marks the proper solution:

<img src="{{ site.baseurl }}/images/python-dividecurvesearching.svg" width="100%" float="right">

In the example above, the curve has been divided into equal length segments of 5.0 units each. The red point (the second item in the collection) is the answer we're looking for. All the other points are of no use to us, and you can imagine that the shorter the distance we're looking for, the more redundant points we get. Under normal circumstances I would not think twice and simply use the *rs.DivideCurveLength()* method. However, I'll take this opportunity to introduce you to one of the most ubiquitous, popular and prevalent algorithms in the field of programming today: binary searching.

Imagine you have a list of integers which is -say- ten thousand items long and you want to find the number closest to sixteen. If this list is unordered (as opposed to sorted) , like so:

> {-2, -10, 12, -400, 80, 2048, 1, 10, 11, -369, 4, -500, 1548, 8, … , 13, -344}

you have pretty much no option but to compare every item in turn and keep a record of which one is closest so far. If the number sixteen doesn't occur in the list at all, you'll have to perform ten thousand comparisons before you know for sure which number was closest to sixteen. This is known as a worst-case performance, the best-case performance would be a single comparison since sixteen might just happen to be the first item in the list... if you're lucky.

The method described above is known as a list-search and it is a pretty inefficient way of searching a large dataset and since searching large datasets is something that we tend to do a lot in computer science, plenty research has gone into speeding things up. Today there are so many different search algorithms that we've had to put them into categories in order to keep a clear overview. However, pretty much all efficient searching algorithms rely on the input list being sorted, like so:

> {-500, -400, -369, -344, -10, -2, 1, 4, 8, 10, 11, 12, 13, 80, … , 1548, 2048}

Once we have a sorted list it is possible to improve our worst case performance by orders of magnitude. For example, consider a slightly more advanced list-search algorithm which aborts the search once results start to become worse. Like the original list-search it will start at the first item {-500}, then continue to the second item {-400}. Since {-400} is closer to sixteen than {-500}, there is every reason to believe that the next item in the list is going to be closer still. This will go on until the algorithm has hit the number thirteen. Thirteen is already pretty close to sixteen but there is still some wiggle room so we cannot be absolutely sure ({14; 15; 16; 17; 18} are all closer and {19} is equally close). However, the next number in the list is {80} which is a much, worse result than thirteen. Now, since this list is sorted we can be sure that every number after {80} is going to be worse still so we can safely abort our search knowing that thirteen is the closest number. Now, if the number we're searching for is near the beginning of the list, we'll have a vast performance increase, if it's near the end, we'll have a small performance increase. On average though, the sorted-list-search is twice as fast as the old-fashioned-list-search.

Binary-searching does far better. Let us return to our actual problem to see how binary-searching works; find the point on a curve that marks a specific length along the curve. In the image below, the point we are looking for has been indicated with a small yellow tag, but of course we don't know where it is when we begin our search. Instead of starting at the beginning of the curve, we start halfway between {tmin} and {tmax} (halfway the domain of the curve). Since we can ask Rhino what the length is of a certain curve subdomain we can calculate the length from {tmin} to {1}. This happens to be way too much, we're looking for something less than half this length. Thus we divide the bit between {tmin} and {1} in half yet again, giving us {2}. We again measure the distance between {tmin} and {2}, and see that again we're too high, but this time only just. We keep on dividing the remainder of the domain in half until we find a value {6} which is close enough for our purposes:


<img src="{{ site.baseurl }}/images/primerbinarycurvesearching.svg" width="100%" float="right">

This is an example of the simplest implementation of a binary-search algorithm and the performance of binary searching is O(log n) which is a fancy way of saying that it's fast. Really, really fast. And what's more, when we enlarge the size of the collection we're searching, the time taken to find an answer doesn't increase in a similar fashion (as it does with list-searching). Instead, it becomes relatively faster and faster as the size of the collection grows. For example, if we double the size of the array we're searching to 20,000 items, a list-search algorithm will take twice as long to find the answer, whereas a binary-searcher only takes ~1.075 times as long.

The theory of binary searching might be easy to grasp (maybe not right away, but you'll see the beauty eventually), any practical implementation has to deal with some annoying, code-bloating aspects. For example, before we start a binary search operation, we must make sure that the answer we're looking for is actually contained within the set. In our case, if we're looking for a point {P} on the curve {C} which is 100.0 units away from the start of {C}, there exists no answer if {C} is shorter than 100.0 itself. Also, since we're dealing with a parameter domain as opposed to a list of integers, we do not have an actual list showing all the possible values. This array would be too big to fit in the memory of your computer. Instead, all we have is the knowledge that any number between and including {tmin} and {tmax} is theoretically possible. Finally, there might not exist an exact answer. All we can really hope for is that we can find an answer within tolerance of the exact length. Many operations in computational geometry are tolerance bound, sometimes because of speed issues (calculating an exact answer would take far too long), sometimes because an exact answer cannot be found (there is simply no math available, all we can do is make a set of guesses each one progressively better than the last).

At any rate, here's the binary-search script I came up with, I'll deal with the inner workings afterwards:

```python
def BSearchCurve(idCrv, Length, Tolerance):
    Lcrv = rs.CurveLength(idCrv)
    if Lcrv<Length: return

    tmin = rs.CurveDomain(idCrv)[0]
    tmax = rs.CurveDomain(idCrv)[1]
    t0 = tmin
    t1 = tmax
    while True:
        t = 0.5*(t1+t0)
        Ltmp = rs.CurveLength(idCrv, 0, [tmin, t])
        if abs(Ltmp-Length)<Tolerance: break
        if Ltmp<Length: t0=t
        else: t1 = t
    return t
```


<table rules="rows">
<tr>
<th>Line</th>	
<th>Description</th>
</tr>
<tr>
<td style="vertical-align:top;text-align:right;padding:0px 10px;">1</td>
<td>Note that this is not a complete script, it is only the search function. The complete script is supplied in the article archive. This function takes a curve ID, a desired length and a tolerance. The return value is <i>None</i> if no solution exists (i.e. if the curve is shorter than <i>Length</i>) or otherwise the parameter that marks the desired length.</td>
</tr>
<tr>
<td style="vertical-align:top;text-align:right;padding:0px 10px;">2</td>
<td>Ask Rhino for the total curve length.
</td>
</tr>
<tr>
<td style="vertical-align:top;text-align:right;padding:0px 10px;">3</td>
<td>Make sure the curve is longer than <i>Length</i>. If it isn't, abort.</td>
</tr>
<tr>
<td style="vertical-align:top;text-align:right;padding:0px 10px;">5...6</td>
<td>Store the minimum and maximum parameters of this curve domain. If you're confused about me calling the <i>Rhino.CurveDomain()</i> function twice instead of just once and store the resulting array, you may congratulate yourself. It would indeed be faster to not call the same method twice in a row. However, since lines 7 and 8 are not inside a loop, they will only execute once which reduces the cost of the penalty. 99% of the time spend by this function is because of lines 16~25, if we're going to be zealous about speed, we should focus on this part of the code.</td>
</tr>
<tr>
<td style="vertical-align:top;text-align:right;padding:0px 10px;">7...8</td>
<td><i>t0</i>, <i>t1</i> and <i>t</i> will be the variables used to define our current subdomain. <i>t0</i> will mark the lower bound and <i>t1</i> the upper bound. <i>t</i> will be halfway between <i>t0</i> and <i>t1</i>. We need to start with the whole curve in mind, so <i>t0</i> and <i>t1</i> will be similar to <i>tmin</i> and <i>tmax</i>.</td>
</tr>
<tr>
<td style="vertical-align:top;text-align:right;padding:0px 10px;">9</td>
<td>Since we do not know in advance how many steps our binary searcher is going to take, we have to use an infinite loop.</td>
</tr>
<tr>
<td style="vertical-align:top;text-align:right;padding:0px 10px;">10</td>
<td>Calculate <i>t</i> always exactly in the middle of {<i>t0</i>, <i>t1</i>}.</td>
</tr>
<tr>
<td style="vertical-align:top;text-align:right;padding:0px 10px;">11</td>
<td>Calculate the length of the subcurve from the start of the curve (<i>tmin</i>) to our current parameter (<i>t</i>).</td>
</tr>
<tr>
<td style="vertical-align:top;text-align:right;padding:0px 10px;">12</td>
<td>If this length is close enough to the desired length, then we are done and we can abort the infinite loop. <i>abs()</i> -in case you were wondering- is a Python function that returns the absolute (non-negative) value of a number. This means that the <i>tolerance</i> argument works equally strong in both directions, which is what you'd usually want.</td>
</tr>
<tr>
<td style="vertical-align:top;text-align:right;padding:0px 10px;">13...14</td>
<td>This is the magic bit. Looks harmless enough doesn't it? 
What we do here is adjust the subdomain based on the result of the length comparison. If the length of the subcurve {<i>tmin</i>, <i>t</i>} is shorter than <i>Length</i>, then we want to restrict ourself to the lower half of the old subdomain. If, on the other hand, the subcurve length is shorter than <i>Length</i>, then we want the upper half of the old domain. 
Notice how much more compact programming code is compared to English?</td>
</tr>
<tr>
<td style="vertical-align:top;text-align:right;padding:0px 10px;">15</td>
<td>Return the solved <i>t</i>-parameter.</td>
</tr>
</table>

I have unleashed this function on a smooth curve with a fairly well distributed parameter space (i.e. no sudden jumps in parameter "density") and the results are listed below. The length of the total curve was 200.0 mm and I wanted to find the parameter for a subcurve length of 125.0 mm. My tolerance was set to 0.0001 mm. As you can see it took 18 refinement steps in the *BSearchCurve()* function to find an acceptable solution. Note how fast this algorithm homes in on the correct value, after just 6 steps the remaining error is less than 1%. Ideally, with every step the accuracy of the guess is doubled, in practise however you're unlikely to see such a neat progression. In fact, if you closely examine the table, you'll see that sometimes the new guess overshoots the solution so much it actually becomes worse than before (like between steps #9 and #10). 

I've greyed out the subdomain bound parameters that remained identical between two adjacent steps. You can see that sometimes multiple steps in the same direction are required.

<img src="{{ site.baseurl }}/images/primer-subdivisionchart.svg" width="100%" float="right">

Now for the rest of the script as outlines on page 78:

```python
def equidistanceoffset():
    srf_id = rs.GetObject("Pick surface to offset", 8, True, True)
    if not srf_id: return

    offset = rs.GetReal("Offset distance", 1.0, 0.0)
    if not offset: return

    udomain = rs.SurfaceDomain(srf_id, 0)
    ustep = (udomain[1]-udomain[0])/200
    rs.EnableRedraw(False)

    offsetvertices = []
    u = udomain[0]
    while u<=(udomain[1]+0.5*ustep):
        isocurves = rs.ExtractIsoCurve(srf_id, (u,0), 1)
        if isocurves:
            t = BSearchCurve(isocurves[0], offset, 0.001)
            if t is not None:
                offsetvertices.append(rs.EvaluateCurve(isocurves[0], t))
            rs.DeleteObjects(isocurves)
        u+=ustep
        
    if offsetvertices: rs.AddInterpCrvOnSrf(srf_id, offsetvertices)
    rs.EnableRedraw(True)
```


<table>
<tr>
<td>
If I've done my job so far, the above shouldn't require any explanation. All of it is straight forward scripting code.
<br><br>
The image on the right shows the result of the script, where offset values are all multiples of 10. The dark green lines across the green strip (between offsets 80.0 and 90.0)  are all exactly 10.0 units long. 
</td>
<td width="40%"><img src="{{ site.baseurl }}/images/primer-equidistantoffset-result.svg" width="100%" height="300" float="right"></td>
</tr>
</table>

### 8.7.3 Geometric curve properties

Since curves are geometric objects, they possess a number of properties or characteristics which can be used to describe or analyze them. For example, every curve has a starting coordinate and every curve has an ending coordinate. When the distance between these two coordinates is zero, the curve is closed. Also, every curve has a number of control-points, if all these points are located in the same plane, the curve as a whole is planar. Some properties apply to the curve as a whole, others only apply to specific points on the curve. For example, planarity is a global property while tangent vectors are a local property. Also, some properties only apply to some curve types. So far we've dealt with lines, polylines, circles, ellipses, arcs and nurbs curves:

<img src="{{ site.baseurl }}/images/primer-curvetypes.svg" width="100%" float="right">

The last available curve type in Rhino is the polycurve, which is nothing more than an amalgamation of other types. A polycurve can be a series of line curves for example, in which case it behaves similarly to a polyline. But it can also be a combination of lines, arcs and nurbs curves with different degrees. Since all the individual segments have to touch each other (G0 continuity is a requirement for polycurve segments), polycurves cannot contain closed segments. However, no matter how complex the polycurve, it can always be represented by a nurbs curve. All of the above types can be represented by a nurbs curve.

The difference between an actual circle and a nurbs-curve-that-looks-like-a-circle is the way it is stored. A nurbs curve doesn't have a Radius property for example, nor a Plane in which it is defined. It is possible to reconstruct these properties by evaluating derivatives and tangent vector and frames and so on and so forth, but the data isn't readily available. In short, nurbs curves lack some global properties that other curve types do have. This is not a big issue, it's easy to remember what properties a nurbs curve does and doesn't have. It is much harder to deal with local properties that are not continuous. For example, imagine a polycurve which has a zero-length line segment embedded somewhere inside. The t-parameter at the line beginning is a different value from the t-parameter at the end, meaning we have a curve subdomain which has zero length. It is impossible to calculate a normal vector inside this domain:

<img src="{{ site.baseurl }}/images/primer-polycurvecompound.svg" width="100%" float="right">

This polycurve consists of five curve segments (a nurbs-curve, a zero-length line-segment, a proper line-segment, a 90° arc and another nurbs-curve respectively) all of which touch each other at the indicated t-parameters. None of them are tangency continuous, meaning that if you ask for the tangent at parameter {t<sub>3</sub>}, you might either get the tangent at the end of the purple segment or the tangent at the beginning of the green segment. However, if you ask for the tangent vector halfway between {t<sub>1</sub>} and {t<sub>2</sub>}, you get nothing. The curvature data domain has an even bigger hole in it, since both line-segments lack any curvature:

<img src="{{ site.baseurl }}/images/primer-polycurvelocalevaluation.svg" width="100%" float="right">

When using curve properties such as tangents, curvature or perp-frames, we must always be careful to not blindly march on without checking for property discontinuities. An example of an algorithm that has to deal with this would be the *_CurvatureGraph* in Rhino. It works on all curve types, which means it must be able to detect and ignore linear and zero-length segments that lack curvature.

One thing the *_CurvatureGraph* command does not do is insert the curvature graph objects, it only draws them on the screen. We're going to make a script that inserts the curvature graph as a collection of lines and interpolated curves. We'll run into several issues already outlined in this paragraph.

In order to avoid some *G* continuity problems we're going to tackle the problem span by span. In case you haven't suffered left-hemisphere meltdown yet; the shape of every knot-span is determined by a certain mathematical function known as a polynomial and is (in most cases) completely smooth. A span-by-span approach means breaking up the curve into its elementary pieces, as shown on the left:

<img src="{{ site.baseurl }}/images/primer-polycurvecurvaturegraph.svg" width="100%" float="right">

This is a polycurve object consisting of seven pieces; lines {A; C; E}, arcs {B; D} and nurbs curves {F; G}. When we convert the polycurve to a nurbs representation we get a degree 5 nurbs curve with 62 pieces (knot-spans). Since this curve was made by joining a bunch of other curves together, there are kinks between all individual segments. A kink is defined as a grouping of identical knots on the interior of a curve, meaning that the curve actually intersects one of its interior control-points. A kink therefore has the potential to become a sharp crease in an otherwise smooth curve, but in our case all kinks connect segments that are G1 continuous. The kinks have been marked by white circles in the image on the right. As you can see there are also kinks in the middle of the arc segments {B; D}, which were there before we joined the curves together. In total this curve has ten kinks, and every kink is a grouping of five similar knot parameters (this is a D<sup>5</sup> curve). Thus we have a sum-total of 40 zero-length knot-spans. Never mind about the math though, the important thing is that we should prepare for a bunch of zero-length spans so we can ignore them upon confrontation.

The other problem we'll get is the property evaluation issue I talked about on the previous page. On the transition between knots the curvature data may jump from one value to another. Whenever we're evaluating curvature data near knot parameters, we need to know if we're coming from the left or the right.

I'm sure all of this sounds terribly complicated. In fact, I'm sure it is terribly complicated, but these things should start to make sense. It is no longer enough to understand how scripts work under ideal circumstances, by now, you should understand why there are no ideal circumstances and how that affects programming code.

Since we know exactly what we need to do in order to mimic the *_CurvatureGraph* command, we might as well start at the bottom. The first thing we need is a function that creates a curvature graph on a subcurve, then we can call this function with the knot parameters as sub-domains in order to generate a graph for the whole curve:

<img src="{{ site.baseurl }}/images/primer-spancurvaturegraph.svg" width="100%" float="right">

Our function will need to know the ID of the curve in question, the subdomain {t<sub>0</sub>; t<sub>1</sub>}, the number of samples it is allowed to take in this domain and the scale of the curvature graph. The return value should be a collection of object IDs which were inserted to make the graph. This means all the perpendicular red segments and the  dashed black curve connecting them.

```python
def addcurvaturegraphsection(idCrv, t0, t1, samples, scale):
    if (t1-t0)<=0.0: return
    tstep = (t1-t0)/samples
    points = []
    objects = []
    for t in rs.frange(t0,t1+(0.5*tstep),tstep):
        if t>=t1:t = t1-1e-10
        cData = rs.CurveCurvature(idCrv, t)
        if not cData:
            points.append(rs.EvaluateCurve(idCrv, t))
        else:
            c = rs.VectorScale(cData[4], scale)
            a = cData[0]
            b = rs.VectorSubtract(a, c)
            objects.append(rs.AddLine(a,b))
            points.append(b)

    objects.append(rs.AddInterpCurve(points))
    return objects
```


<table rules="rows">
<tr>
<th>Line</th>	
<th>Description</th>
</tr>
<tr>
<td style="vertical-align:top;text-align:right;padding:0px 10px;">2</td>
<td>Check for a null span, this happens inside kinks.</td>
</tr>
<tr>
<td style="vertical-align:top;text-align:right;padding:0px 10px;">3</td>
<td>Determine a step size for our loop (Subdomain length / Sample count).
</td>
</tr>
<tr>
<td style="vertical-align:top;text-align:right;padding:0px 10px;">5</td>
<td><i>objects()</i> will hold the IDs of the perpendicular lines, and the connecting curve.</td>
</tr>
<tr>
<td style="vertical-align:top;text-align:right;padding:0px 10px;">6</td>
<td>Define the loop and make sure we always process the final parameter by increasing the threshold with half the step size.</td>
</tr>
<tr>
<td style="vertical-align:top;text-align:right;padding:0px 10px;">7</td>
<td>Make sure <i>t</i> does not go beyond <i>t1</i>, since that might give us the curvature data of the next segment.</td>
</tr>
<tr>
<td style="vertical-align:top;text-align:right;padding:0px 10px;">10</td>
<td>In case of a curvature data discontinuity, do not add a line segment but append the point on the curve at the current curve coordinate <i>t</i>.</td>
</tr>
<tr>
<td style="vertical-align:top;text-align:right;padding:0px 10px;">12...16</td>
<td>Compute the A and B coordinates, append them to the appropriate array and add the line segment.</td>
</tr>
</table>

Now, we need to write a utility function that applies the previous function to an entire curve. There's no rocket science here, just an iteration over the knot-vector of a curve object:

```primer
def addcurvaturegraph( idCrv, spansamples, scale):
    allGeometry = []
    knots = rs.CurveKnots(idCrv)
    p=5
    for i in range(len(knots)-1):
        tmpGeometry = addcurvaturegraphsection(idCrv, knots[i], knots[i+1], spansamples, scale)
        if tmpGeometry: allGeometry.append(tmpGeometry)
    rs.AddObjectsToGroup(allGeometry, rs.AddGroup())
    return allGeometry
```

<table rules="rows">
<tr>
<th>Line</th>	
<th>Description</th>
</tr>
<tr>
<td style="vertical-align:top;text-align:right;padding:0px 10px;">2</td>
<td><i>allGeometry</i> will be a list of all IDs generated by repetitive calls to <i>AddCurvatureGraphSection()</i></td>
</tr>
<tr>
<td style="vertical-align:top;text-align:right;padding:0px 10px;">3</td>
<td><i>knots</i> is the knot vector of the nurbs representation of <i>idCrv</i>.
</td>
</tr>
<tr>
<td style="vertical-align:top;text-align:right;padding:0px 10px;">5</td>
<td>We want to iterate over all knot spans, meaning we have to iterate over all (except the last) knot in the knot vector. Hence the minus one at the end.</td>
</tr>
<tr>
<td style="vertical-align:top;text-align:right;padding:0px 10px;">6</td>
<td>Place a call to <i>addcurvaturegraphsection()</i> and store all resulting IDs in <i>tmpGeometry</i>.</td>
</tr>
<tr>
<td style="vertical-align:top;text-align:right;padding:0px 10px;">7</td>
<td>If the result of <i>AddCurvatureGraphSection()</i> is not <i>Null</i>, then append all items in <i>tmpGeometry</i> to <i>allGeometry</i>.</td>
</tr>
<tr>
<td style="vertical-align:top;text-align:right;padding:0px 10px;">8</td>
<td>Put all created objects into a new group.</td>
</tr>
</table>

The last bit of code we need to write is a bit more extensive than we've done so far. Until now we've always prompted for a number of values before we performed any action. It is actually far more user-friendly to present the different values as options in the command line while drawing a preview of the result.

UI code tends to be very beefy, but it rarely is complex. It's just irksome to write because it always looks exactly the same. In order to make a solid command-line interface for your script you have to do the following:

- Reserve a place where you store all your preview geometry
- Initialize all settings with sensible values
- Create all preview geometry using the default settings
- Display the command line options
- Parse the result (be it escape, enter or an option or value string)
- Select case through all your options
- If the selected option is a setting (as opposed to options like "Cancel" or "Accept") then display a prompt for that setting
- Delete all preview geometry
- Generate new preview geometry using the changed settings.

```python
def createcurvaturegraph():
    curve_ids = rs.GetObjects("Curves for curvature graph", 4, False, True, True)
    if not curve_ids: return

    samples = 10
    scale = 1.0

    preview = []
    while True:
        rs.EnableRedraw(False)
        for p in preview: rs.DeleteObjects(p)
        preview = []
        for id in curve_ids:
            cg = addcurvaturegraph(id, samples, scale)
            preview.append(cg)
        rs.EnableRedraw(True)

        result = rs.GetString("Curvature settings", "Accept", ("Samples", "Scale", "Accept"))
        if not result:
            for p in preview: rs.DeleteObjects(p)
            break
        result = result.upper()
        if result=="ACCEPT": break
        elif result=="SAMPLES":
            numsamples = rs.GetInteger("Number of samples per knot-span", samples, 3, 100)
            if numsamples: samples = numsamples
        elif result=="SCALE":
            sc = rs.GetReal("Scale of the graph", scale, 0.01, 1000.0)
            if sc: scale = sc
```


<table rules="rows">
<tr>
<th>Line</th>	
<th>Description</th>
</tr>
<tr>
<td style="vertical-align:top;text-align:right;padding:0px 10px;">2</td>
<td>Prompt for any number of curves, we do not want to limit our script to just one curve.</td>
</tr>
<tr>
<td style="vertical-align:top;text-align:right;padding:0px 10px;">5...6</td>
<td>Our default values are a scale factor of 1.0 and a span sampling count of 10.
</td>
</tr>
<tr>
<td style="vertical-align:top;text-align:right;padding:0px 10px;">8</td>
<td><i>preview()</i> is a list that contains arrays of IDs. One for each curve in <i>idCurves</i>.</td>
</tr>
<tr>
<td style="vertical-align:top;text-align:right;padding:0px 10px;">9</td>
<td>Since users are allowed to change the settings an infinite number of times, we need an infinite loop around our UI code.</td>
</tr>
<tr>
<td style="vertical-align:top;text-align:right;padding:0px 10px;">10...11</td>
<td>First of all, delete all the preview geometry, if present.</td>
</tr>
<tr>
<td style="vertical-align:top;text-align:right;padding:0px 10px;">13...15</td>
<td>Then, insert all the new preview geometry.</td>
</tr>
<tr>
<td style="vertical-align:top;text-align:right;padding:0px 10px;">28</td>
<td>Once the new geometry is in place, display the command options. The array at the end of the <i>rs.GetString()</i> method is a list of command options that will be visible.</td>
</tr>
<tr>
<td style="vertical-align:top;text-align:right;padding:0px 10px;">19...21</td>
<td>If the user aborts (pressed Escape), we have to delete all preview geometry and exit the sub.</td>
</tr>
<tr>
<td style="vertical-align:top;text-align:right;padding:0px 10px;">23...29</td>
<td>If the user clicks on an option, <i>result</i> will be the option name. The best method IronPython implements to treat the choice is the <i>If...Then</i> statement shown.</td>
</tr>
<tr>
<td style="vertical-align:top;text-align:right;padding:0px 10px;">23</td>
<td>In the case of "Accept", all we have to do is exit the sub without deleting the preview geometry.</td>
</tr>
<tr>
<td style="vertical-align:top;text-align:right;padding:0px 10px;">24...26</td>
<td>If the picked option was "Samples", then we have to ask the user for a new sample count. If the user pressed Escape during this nested prompt, we do not abort the whole script (typical Rhino behaviour would dictate this), but instead return to the base prompt.</td>
</tr>
<tr>
<td style="vertical-align:top;text-align:right;padding:0px 10px;">27...29</td>
<td>If the picked option was "Scale", then we have to ask the user for a new scale factor, and . If the user pressed Escape during this nested prompt, we do not abort the whole script (typical Rhino behaviour would dictate this), but instead return to the base prompt.</td>
</tr>
</table>



---

#### Related Topics

- [Where to find help - Next Topic >>]({{ site.baseurl }}/guides/rhinopython/primer-101/1-2-where-to-find-help/)
- [Rhino.Python Primer 101]({{ site.baseurl }}/guides/rhinopython/primer-101/rhinopython101)
- [Running Scripts]({{ site.baseurl }}/guides/rhinopython/python-running-scripts)
- [Canceling Scripts]({{ site.baseurl }}/guides/rhinopython/python-canceling-scripts)
- [Editing Scripts]({{ site.baseurl }}/guides/rhinopython/python-editing-scripts)
- [Scripting Options]({{ site.baseurl }}/guides/rhinopython/python-scripting-options)
- [Reinitializing Python]({{ site.baseurl }}/guides/rhinopython/python-scripting-reinitialize)