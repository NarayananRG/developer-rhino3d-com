---
title: Calculate Radius of Curvature
description: Demonstrates how to compute the radius of curvature of a curve object at a selected location.
authors: ['Dale Fugier']
author_contacts: ['dale']
apis: ['C/C++']
languages: ['C/C++']
platforms: ['Windows']
categories: ['Curves']
origin: http://wiki.mcneel.com/developer/sdksamples/radiusofcurvature
order: 1
keywords: ['rhino']
layout: code-sample-cpp
---

```cpp
CRhinoCommand::result CCommandTest::RunCommand( const CRhinoCommandContext& context )
{
  CRhinoGetObject go;
  go.SetCommandPrompt( L"Select curve for curvature measurement" );
  go.SetGeometryFilter( CRhinoGetObject::curve_object );
  go.GetObjects( 1, 1 );
  if( go.CommandResult() != success )
    return go.CommandResult();

  const ON_Curve* crv = go.Object(0).Curve();
  if( 0 == crv )
    return failure;

  CRhinoGetPoint gp;
  gp.SetCommandPrompt( L"Select point on curve for curvature measurement" );
  gp.Constrain( *crv );
  gp.GetPoint();
  if( gp.CommandResult() != success )
    return gp.CommandResult();

  ON_3dPoint pt = gp.Point();

  double t = 0.0;
  if( !crv->GetClosestPoint(pt, &t) )
  {
    RhinoApp().Print( L"Failed to compute radius of curvature.\n" );
    return failure;
  }

  ON_3dVector tangent = crv->TangentAt( t );
  if( tangent.IsTiny() )
  {
    RhinoApp().Print( L"Failed to compute radius of curvature. Curve may have stacked control points.\n" );
    return failure;
  }

  ON_3dVector curvature = crv->CurvatureAt( t );
  const double k = curvature.Length();
  if( k < ON_SQRT_EPSILON )
  {
    RhinoApp().Print( L"Radius of curvature: infinite.\n" );
    return failure;
  }

  ON_3dVector radius_vector = curvature / (k * k);
  ON_Circle circle;
  if ( !circle.Create(pt, tangent, pt + 2.0 * radius_vector) )
  {
    RhinoApp().Print( L"Failed to compute radius of curvature.\n" );
    return failure;
  }

  context.m_doc.AddCurveObject( circle );
  context.m_doc.AddPointObject( pt );
  context.m_doc.Redraw();

  ON_wString wRadius;
  RhinoFormatNumber( circle.Radius(), wRadius );
  RhinoApp().Print( L"Radius of curvature: %s.\n", wRadius );

  return success;
}
```