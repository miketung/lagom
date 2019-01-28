---
layout: post 
title: Tiny Raytracer in Javascript
categories: blog
---

This is a Javascript implementation of the tutorial [Understandable RayTracing in 256 lines of bare C++](https://github.com/ssloy/tinyraytracer/wiki).

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

    class Material {
        constructor(refract, albedo, color, specular){
            this.refractive_index = refract;
            this.albedo = albedo;  
            this.diffuseColor = color;
            this.specularExponent = specular;
        }
    }

    class Sphere {
        constructor(center, radius, material){
            this.center = center;
            this.radius = radius;
            this.material = material
        }
        ray_intersect(orig, dir) {
            // See http://www.lighthouse3d.com/tutorials/maths/ray-sphere-intersection/
            var L = this.center.subtract(orig);
            var tca = L.dot(dir);
            var d2 = L.dot(L) - (tca * tca);
            if (d2 > this.radius * this.radius) return false; // ray beyond radius
            var thc = Math.sqrt(this.radius * this.radius - d2);
            var t0 = tca - thc;
            var t1 = tca + thc;
            if (t0 < 0) t0 = t1;
            if (t0 < 0) return false;
            return t0;
        }
    }

    class Light {
        constructor(position, intensity) {
            this.position = position;
            this.intensity = intensity;
        }
    }

    function reflect(I, N){
        return I.subtract(N.multiply(2.0 * I.dot(N)));
    }

    function refract(I, N, refractive_index) { // Snell's law
        var cosi = -1 * Math.max(-1.0, Math.min(1.0, I.dot(N)));
        var etai = 1, etat = refractive_index;
        var n = N;
        if (cosi < 0) {
            cosi = -1 * cosi;
            etai = refractive_index;
            etat = 1;
            n = N.multiply(-1);
        }
        var eta = etai / etat;
        var k = 1 - eta*eta*(1-cosi*cosi);
        return k < 0 ? new Vec3(0,0,0) : I.multiply(eta).add(n.multiply(eta * cosi - Math.sqrt(k)));
    }

    // If ray from orig->dir intersect with spheres, return [closest_sphere, hit, N]
    function scene_intersect(orig, dir, spheres) {
        var closest_dist = Number.MAX_VALUE;
        var closest_material = null;
        var hit;
        var N;

        for(var i=0;i<spheres.length;i++) { // Spheres
            var dist_i = spheres[i].ray_intersect(orig, dir);
            if (dist_i && dist_i < closest_dist) {
                closest_dist = dist_i;
                closest_material = spheres[i].material;
                hit = orig.add(dir.multiply(dist_i));
                N = hit.subtract(spheres[i].center).normalize();
            }
        }

        if (Math.abs(dir.y)>1e-3)  {  // Checkerboard
            var d = -(orig.y+4)/dir.y;  //  plane has equation y = -4
            var pt = orig.add(dir.multiply(d));
            if (d>0 && Math.abs(pt.x)<10 && pt.z<-10 && pt.z>-30 && d<closest_dist) {
                closest_dist = d;
                hit = pt;
                N = new Vec3(0,1,0);
                var checkerboard_color = (Math.floor(.5*hit.x+1000) + Math.floor(.5*hit.z)) & 1 ? new Vec3(1,1,1) : new Vec3(1, .7, .3);
                closest_material = new Material(1, [1, 0, 0, 0], checkerboard_color.multiply(0.3), 0);
            }
        }

        if (closest_material)
            return [closest_material, hit, N];
        return false;
    }
    
    function cast_ray (orig, dir, spheres, lights, recurse_depth) {
        if (recurse_depth>4) {
            return background_color; // background color
        }
        var tuple = scene_intersect(orig, dir, spheres);
        // If no intersection, or max recursion reached
        if (!tuple) {
            return background_color; // background color
        }
        
        var material = tuple[0], hit = tuple[1], N = tuple[2];
        
        // reflection, refraction
        var reflect_dir = reflect(dir, N).normalize();
        var refract_dir = refract(dir, N, material.refractive_index).normalize();
        var reflect_orig = reflect_dir.dot(N) < 0 ? hit.subtract(N.multiply(1e-3)) : hit.add(N.multiply(1e-3));  // offset the original point to avoid occlusion by the object itself
        var refract_orig = refract_dir.dot(N) < 0 ? hit.subtract(N.multiply(1e-3)) : hit.add(N.multiply(1e-3));
        var reflect_color = cast_ray(reflect_orig, reflect_dir, spheres, lights, recurse_depth+1);
        var refract_color = cast_ray(refract_orig, refract_dir, spheres, lights, recurse_depth+1);

        
        var diffuse_light_intensity = 0, specular_light_intensity = 0;
        for(var i=0;i<lights.length;i++) {
            var light_dir = lights[i].position.subtract(hit).normalize();
            var light_distance = lights[i].position.subtract(hit).norm();

            //shadows
            var shadow_orig = light_dir.dot(N) < 0 ? hit.subtract(N.multiply(1e-3)) : hit.add(N.multiply(1e-3));
            var shadow_intersect = scene_intersect(shadow_orig, light_dir, spheres);
            if (shadow_intersect && shadow_intersect[1].subtract(shadow_orig).norm() < light_distance)
                continue;   // in the shadow of another object, skip this light source

            diffuse_light_intensity += lights[i].intensity * Math.max(0, light_dir.dot(N));
            specular_light_intensity += Math.pow(Math.max(0, reflect(light_dir.multiply(-1), N).multiply(-1).dot(dir) ),                                              material.specularExponent) * lights[i].intensity;
        }

        //  color is diffuseColor + specular + reflections
        return material.diffuseColor.multiply(diffuse_light_intensity * material.albedo[0]) 
                     .add(new Vec3(1, 1, 1).multiply(specular_light_intensity * material.albedo[1])) 
                     .add(reflect_color.multiply(material.albedo[2]))
                     .add(refract_color.multiply(material.albedo[3]));
    }
    
    function render(origin, spheres, lights) {
        var t0 = new Date().getTime();
        var fov = Math.PI / 2; // field of view
        var tan_fov = Math.tan(fov / 2);
        for (var j = 0; j < height; j++) {
            for (var i = 0; i < width; i++) {
                // compute direction from origin to i,j
                var x = (2* (i + 0.5) / width- 1) * tan_fov * width / height;
                var y = -1 * (2*(j + 0.5) / height -1) * tan_fov;
                var dir = new Vec3(x, y, -1).normalize();
                frameBuffer[j*width + i] = cast_ray(origin, dir, spheres, lights, 0);
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

        console.log("Raytracing took: " + (t1-t0));
        console.log("Drawing took: " + (t2 - t1));
    }

    // Render spheres   
    var ivory = new Material(1.0, [0.6, 0.3, 0.1, 0], new Vec3(0.4, 0.4, 0.3), 50);
    var glass = new Material(1.5, [0.0, 0.5, 0.1, 0.8], new Vec3(0.6, 0.7, 0.8), 125);
    var red_rubber = new Material(1.0, [0.9, 0.1, 0, 0], new Vec3(0.3, 0.1, 0.1), 10);
    var mirror = new Material(1.0, [0, 10, 0.8, 0], new Vec3(1, 1, 1), 1425);
    var background_color = new Vec3(0.2, 0.7, 0.8);
    
    var spheres = [
        new Sphere(new Vec3(-3, 0, -16), 2, ivory),
        new Sphere(new Vec3(3, 0.2, -10), 1, ivory),
        new Sphere(new Vec3(-1, -1.5, -12), 2, glass),
        new Sphere(new Vec3(1.5, -0.5, -18), 3, red_rubber),
        new Sphere(new Vec3(1.5, -0.5, -18), 3, red_rubber),
        new Sphere(new Vec3(7, 5, -18), 4, mirror)
    ];
    var lights = [
        new Light(new Vec3(-20, 20,  20), 1.5),
        new Light(new Vec3(30, 50, -25), 1.8),
        new Light(new Vec3(30, 20,  30), 1.7)
    ];
    var origin = new Vec3(0,0,0);

    render(origin, spheres, lights);

    // Drag to move origin
    var downX=0, downY = 0;
    var isMouseDown = false;
    canvas.onmousedown = function(e) {
        isMouseDown = true;
        downX = e.pageX - this.offsetLeft;
        downY = e.pageY - this.offsetTop;
    }
    canvas.onmouseup = function(e) {
        isMouseDown = false;
    }
    canvas.onmousemove = function(e) {
        if (isMouseDown) {
            x = e.pageX - this.offsetLeft;
            y = e.pageY - this.offsetTop;
            deltaX =  x - downX;
            deltaY =  y - downY;
            origin = origin.add(new Vec3(deltaX/50, deltaY/50, 0));
            render(origin, spheres, lights);
            downX = x;
            downY = y;
        }
    };
</script>