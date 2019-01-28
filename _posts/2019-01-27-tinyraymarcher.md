---
layout: post 
title: Tiny Raymarcher in Javascript
categories: blog
---

This is a Javascript implementation of the tutorial [TinyKaboom](https://github.com/ssloy/tinykaboom/wiki).

<div class="container">
    <canvas id="canvas1"></canvas>
</div>

<script type="text/javascript" >
    var width   = 640;
    var height  = 480;
    var canvas  = document.getElementById('canvas1');
    canvas.width = width;
    canvas.height = height;
    var ctx     = canvas.getContext('2d');
    var frameBuffer = new Array(width * height);
    var imageData = ctx.createImageData(width, height);

    // Basic Vector Operations
    class Vec3 {
        constructor(x,y,z) {
            this.x = x;
            this.y = y;
            this.z = z;
        }
        times(v) {
            return new Vec3(this.x * v.x, this.y * v.y, this.z * v.z);
        }
        add(v) {
            return new Vec3(this.x + v.x, this.y + v.y, this.z + v.z);
        }
        subtract(v) {
            return new Vec3(this.x - v.x, this.y - v.y, this.z - v.z);
        }
        dot( v) {
            return this.x * v.x + this.y * v.y + this.z * v.z; // returns scalar
        }
        multiply(k) {
            return new Vec3(this.x * k, this.y * k, this.z * k);
        }
        norm() {
            return Math.sqrt(this.x * this.x + this.y * this.y + this.z * this.z);
        }
        normalize() { 
            var n = this.norm();
            if (n == 0) return this;
            return new Vec3(this.x / n, this.y / n, this.z / n);
        }
    }

    var sphere_radius = 1.5;
    var noise_amplitude = 1.0;
    function lerp(v0, v1, t) { // float,float,float -> float
        return v0 + (v1-v0)*Math.max(0, Math.min(1, t));
    }
    function lerpV(v0, v1, t) { // vec3,vec3,float -> Vec3
        return v0.add(v1.subtract(v0).multiply(Math.max(0, Math.min(1, t))));
    }
    function hash(n) { // float -> float
        var x = Math.sin(n)*43758.5453;
        return x-Math.floor(x);
    }
    function noise(x) { //Vec3 -> float
        var p = new Vec3(Math.floor(x.x), Math.floor(x.y), Math.floor(x.z));
        var f = new Vec3(x.x-p.x, x.y-p.y, x.z-p.z);
        f = f.multiply(f.dot(new Vec3(3, 3, 3).subtract(f.multiply(2)) ));
        var n = p.dot(new Vec3(1, 57, 113)); //float
        return lerp(lerp(
                        lerp(hash(n +  0), hash(n +  1), f.x),
                        lerp(hash(n + 57), hash(n + 58), f.x), f.y),
                    lerp(
                        lerp(hash(n + 113), hash(n + 114), f.x),
                        lerp(hash(n + 170), hash(n + 171), f.x), f.y), f.z);
    }
    function rotate(v) { // Vec3 -> Vec3
        return new Vec3(new Vec3(0.00,  0.80,  0.60).dot(v), new Vec3(-0.80,  0.36, -0.48).dot(v), new Vec3(-0.60, -0.48,  0.64).dot(v));
    }
    function fractal_brownian_motion(x) { //Vec3 -> float
        var p = rotate(x);
        var f = 0;
        f += 0.5000*noise(p); p = p.multiply(2.32);
        f += 0.2500*noise(p); p = p.multiply(3.03);
        f += 0.1250*noise(p); p = p.multiply(2.61);
        f += 0.0625*noise(p);
        return f/0.9375;
    }
    var yellow = new Vec3(1.7, 1.3, 1.0); // note that the color is "hot", i.e. has components >1
    var orange = new Vec3(1.0, 0.6, 0.0);
    var red = new Vec3(1.0, 0.0, 0.0);
    var darkgray = new Vec3(0.2, 0.2, 0.2);
    var gray = new Vec3(0.4, 0.4, 0.4);

    function palette_fire(d) {
        var x = Math.max(0, Math.min(1, d));
        if (x<.25)
            return lerpV(gray, darkgray, x*4);
        else if (x<.5)
            return lerpV(darkgray, red, x*4-1);
        else if (x<.75)
            return lerpV(red, orange, x*4-2);
        return lerpV(orange, yellow, x*4-3);
    }
    function signed_distance(p) { // Vec3 -> float. Distance beyond sphere
        var displacement = -3 * Math.random()*noise_amplitude; //-1*fractal_brownian_motion(p.multiply(3.4))*noise_amplitude;
        return p.norm() - (sphere_radius + displacement);
    }
    function sphere_trace(orig, dir) { // returns 3d-position on sphere surface 
        if (orig.dot(orig) - Math.pow(orig.dot(dir), 2) > Math.pow(sphere_radius, 2)) return false; // early discard
        var pos = orig;
        for (var i=0;i<128;i++) {
            var d = signed_distance(pos);
            if (d<0) return pos;
            pos = pos.add(dir.multiply(Math.max(d * 0.1, 0.01))) ; // march
        }
        return false;
    }
    function distance_field_normal(p){ //Vec3 -> Vec3. normal to surface of sphere
        var eps = 0.1;
        var d = signed_distance(p);
        var nx = signed_distance(p.add(new Vec3(eps, 0, 0))) - d;
        var ny = signed_distance(p.add(new Vec3(0, eps, 0))) - d;
        var nz = signed_distance(p.add(new Vec3(0, 0, eps))) - d;
        return new Vec3(nx, ny, nz).normalize();
    }

    function render() {
        var t0 = new Date().getTime();
        var fov = Math.PI / 3.0;
        var origin =new Vec3(0,0,3); // camera origin at [0,0,3], pointing in negative z
        var light =new Vec3(10, 10, 10); // light position
        for(var j=0;j<height;j++) {
            for (var i=0;i<width;i++){
                var dir_x = (i + 0.5) - width/2.0;
                var dir_y = -(j + 0.5) + height/2.0;  //this flips image at same time
                var dir_z = -height/(2.0 * Math.tan(fov / 2.0));
                var hit = sphere_trace(origin, new Vec3(dir_x,dir_y,dir_z).normalize());
                if (hit) {
                    var noise_level = (sphere_radius-hit.norm())/noise_amplitude;
                    var light_dir = light.subtract(hit).normalize(); // direction from light->hit computed by subtraction
                    var light_intensity = Math.max(0.4, light_dir.dot(distance_field_normal(hit))); 
                    frameBuffer[j*width+i] = palette_fire((-0.2 + noise_level)*2).multiply(light_intensity);
                } else {
                    frameBuffer[j*width+i] = new Vec3(0.2, 0.7, 0.8); //background
                }
            }
        }
        
        var t1 = new Date().getTime();
        var data = imageData.data;
        var i=0, j=0; 
        while(i < data.length) {
            data[i] = 255 * frameBuffer[j].x;
            data[i+1] = 255 * frameBuffer[j].y;
            data[i+2] = 255 * frameBuffer[j].z;
            data[i+3] = 255;
            i += 4;
            j += 1;
        }
        ctx.putImageData(imageData, 0, 0);
        var t2 = new Date().getTime();

        console.log("Raymarching took: " + (t1-t0));
        console.log("Drawing took: " + (t2 - t1));
    }

    render();
</script>