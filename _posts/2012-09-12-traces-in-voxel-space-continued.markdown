---
layout: post
title:  "Traces in voxel space continued"
date:   2012-09-12 19:48:00 +0000
categories: programming voxels
---
So [last post]({% post_url 2012-09-11-traces-in-voxel-space %}) I showed how we could get the voxel indices for all the voxels hit by a ray. Today we go further!

While getting the indices for each voxel is nice, what would be even nicer would be to return the world space position that the ray first hits the voxel.
If you remember the tx/ty values we used last time for deciding which direction to step next. Those values have more meaning than just the direction to step. They are the time the ray first hits the next relevent axis plane (that is tx tells us the t such that start+ray*t first crosses into the next voxel in the x axis).
So each time we do a step before incrementing the t we're stepping we can use it to calculate a world space hit point.
{% highlight csharp %}
Point hit = start
int x = p0.x
int y = p0.y
while (true)
{
    yield Pair(Voxel(x, y), hit)

    if (tx <= ty)
    {
        if (x == p1.x) break

        hit = start + dir * tx
        tx += dtx;
        x += dx;
    }
    else
    {
        if (y == p1.y) break

        hit = start + dir * ty
        ty += dty;
        y += dy;
    }
}
{% endhighlight %}
As you can see the first hit point is simply the start point.
It's a simple trick but very effective. Here's an updated version of the SlimDX program that demonstrates this. It draws blue circles at each hit location (skipping the drawing of the first hit because that would just draw over our green circle).

{% highlight csharp %}
using System;
using System.Collections.Generic;
using System.Drawing;
using System.Linq;
using SlimDX.Direct2D;
using SlimDX.Windows;

namespace VoxelTrace
{
    class Form : RenderForm
    {
        public Form()
        {
            ClientSize = new System.Drawing.Size(100 * Program.Lines, 100 * Program.Lines);
            MaximumSize = MinimumSize = Size;
            Text = "Voxel trace";
        }
    }

    class Program
    {
        static Factory Factory;
        static WindowRenderTarget RenderTarget;

        static SolidColorBrush BlackBrush;
        static SolidColorBrush BlueBrush;
        static SolidColorBrush GreenBrush;
        static SolidColorBrush RedBrush;
        static LinearGradientBrush GreenRedBrush;

        static Point P1, P2;

        public const int Lines = 8;

        static void Main(string[] args)
        {
            Form form = new Form();

            Factory = new Factory();

            var renderTargetProperties = new WindowRenderTargetProperties()
            {
                Handle = form.Handle,
                PresentOptions = PresentOptions.Immediately,
                PixelSize = form.ClientSize,
            };

            RenderTarget = new WindowRenderTarget(Factory, renderTargetProperties);

            BlackBrush = new SolidColorBrush(RenderTarget, Color.Black);
            BlueBrush = new SolidColorBrush(RenderTarget, Color.Blue);
            GreenBrush = new SolidColorBrush(RenderTarget, Color.Green);
            RedBrush = new SolidColorBrush(RenderTarget, Color.Red);

            var gradientStops = new GradientStopCollection(RenderTarget, new GradientStop[] {
                new GradientStop() {  Color = Color.Green, Position = 0 },
                new GradientStop() {  Color = Color.Red, Position = 1 }});
            var linearGradientBrushProperties = new LinearGradientBrushProperties()
            {
                StartPoint = P1, EndPoint = P2,
            };
            GreenRedBrush = new LinearGradientBrush(RenderTarget, gradientStops, linearGradientBrushProperties);

            form.MouseClick += new System.Windows.Forms.MouseEventHandler(MouseClick);
            MessagePump.Run(form, Tick);
        }

        static void MouseClick(object sender, System.Windows.Forms.MouseEventArgs e)
        {
            int x = e.X;
            int y = e.Y;

            if (e.Button == System.Windows.Forms.MouseButtons.Left)
                P1 = new Point(x, y);
            if (e.Button == System.Windows.Forms.MouseButtons.Right)
                P2 = new Point(x, y);

            GreenRedBrush.StartPoint = P1;
            GreenRedBrush.EndPoint = P2;

            Trace = Raytrace(P1, P2).ToArray();
        }

        static Tuple<Point, Point>[] Trace;

        static IEnumerable<Tuple<Point, Point>> Raytrace(Point start, Point end)
        {
            int x0 = start.X / 100;
            int y0 = start.Y / 100;

            int x1 = end.X / 100;
            int y1 = end.Y / 100;

            SlimDX.Vector2 dir = new SlimDX.Vector2(end.X - start.X, end.Y - start.Y);

            int dx = Math.Sign(dir.X);
            int dy = Math.Sign(dir.Y);

            double deltax = 100.0 / Math.Abs(dir.X);
            double deltay = 100.0 / Math.Abs(dir.Y);

            long minx = x0 * 100, maxx = minx + 100;
            long miny = y0 * 100, maxy = miny + 100;

            double distx = ((dx >= 0) ? (maxx - start.X) : (start.X - minx)) / 100.0;
            double disty = ((dy >= 0) ? (maxy - start.Y) : (start.Y - miny)) / 100.0;

            double tx = distx * deltax;
            double ty = disty * deltay;

            var prev = new Point(x0, y0);
            var hit = start;
            while (true)
            {
                var grid = new Point(x0, y0);
                yield return Tuple.Create(grid, hit);

                prev = grid;

                if (tx <= ty)
                {
                    if (x0 == x1) break;

                    hit = new Point(
                        start.X + (int)(dir.X * tx),
                        start.Y + (int)(dir.Y * tx));

                    tx += deltax;
                    x0 += dx;
                }
                else
                {
                    if (y0 == y1) break;

                    hit = new Point(
                        start.X + (int)(dir.X * ty),
                        start.Y + (int)(dir.Y * ty));

                    ty += deltay;
                    y0 += dy;
                }
            }
        }

        public static void Tick()
        {
            RenderTarget.BeginDraw();
            RenderTarget.Clear(Color.White);

            for (int i = 0; i < Lines; ++i)
            {
                PointF x0 = new PointF(100 * i, 0), x1 = new PointF(100 * i, 100 * Lines);
                PointF y0 = new PointF(0, 100 * i), y1 = new PointF(100 * Lines, 100 * i);

                RenderTarget.DrawLine(BlackBrush, x0, x1);
                RenderTarget.DrawLine(BlackBrush, y0, y1);
            }

            float radius = 2;
            RenderTarget.FillEllipse(GreenBrush, new Ellipse() { Center = P1, RadiusX = radius, RadiusY = radius });
            RenderTarget.FillEllipse(RedBrush, new Ellipse() { Center = P2, RadiusX = radius, RadiusY = radius });
            RenderTarget.DrawLine(GreenRedBrush, P1, P2);

            if (Trace != null)
            {
                for (int i = 0; i < Trace.Length; ++i)
                {
                    var pair = Trace[i];
                    var p = pair.Item1;
                    var h = pair.Item2;

                    float size = (Trace.Length - i) * 100.0f / Trace.Length;
                    float offset = (100 - size) / 2;

                    var rect = new RectangleF(p.X * 100 + offset, p.Y * 100 + offset, size, size);

                    RenderTarget.DrawRectangle(BlueBrush, rect);

                    if (i != 0)
                        RenderTarget.DrawEllipse(BlueBrush, new Ellipse() { Center = h, RadiusX = radius, RadiusY = radius });
                }
            }

            RenderTarget.EndDraw();
        }
    }
}
{% endhighlight %}