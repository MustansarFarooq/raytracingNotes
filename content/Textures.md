Next we will implement textures. Usually, we think of textures as mapping the point where we hit on an object, to a point on an image. Defining a color at each point of the surface of the object we hit, we refer to our texture and translate that point to get the exact color that maps. To perform a texture lookup, we will need a two-dimensional texture coordinate, which is commonly defined as $(u,v)$. Each pair of $(u,v)$ pairs and maps to a color on the texture.
We will create a texture class, of whose main function, `value()` is to take in this pair $(u,v)$ along with the point where we hit, `p`. 
```cpp
class texture {  
public:  
    virtual ~texture() = default;  
    virtual sf::Vector3f value(double u, double v, const sf::Vector3f& p) const = 0;  
};
```
# Solid Textures
Depending on the type of texture or value function will handle what color to return for that given point. For a texture that is a constant color, all points on our hittable will always yield a fixed color.
```cpp
class solidColor : public texture {  
public:  
    solidColor(sf::Vector3f albedo) : albedo(albedo){};  
  
    solidColor(double r, double g, double b) : solidColor(sf::Vector3f(r,g,b)) {};  
  
    sf::Vector3f value(double u, double v, const sf::Vector3f &p) const override {  
        return albedo;  
    }  
private:  
    sf::Vector3f albedo;  
};
```

## Checkered Textures
Our checkered texture is solid (or spatial) texture. This means that the color depends on where in space the point is. Coordinates $u$ and $v$ don't matter here, but the point at which we hit our sphere does. To simply get a checkered pattern, we can take the point where we hit our sphere, floor each component of that vector, take their sum and compute the result modulo two. Whether it is even or not, dictates if our value will be the even color or the odd color.
```cpp
class checkerTexture : public texture {  
public:  
    checkerTexture(float scale, std::shared_ptr<texture> even, std::shared_ptr<texture> odd) : invScale(1.f/scale), even(even), odd(odd){};  
    checkerTexture(float scale, sf::Vector3f c1, sf::Vector3f c2) : checkerTexture(scale, std::make_shared<solidColor>(c1), std::make_shared<solidColor>(c2)) {};  
    sf::Vector3f value(double u, double v, const sf::Vector3f &p) const override {  
        int x = std::floor(invScale * p.x);  
        int y = std::floor(invScale * p.y);  
        int z = std::floor(invScale * p.z);  
  
        bool isEven = (x+y+z) % 2 == 0;  
  
        return isEven ? even->value(u,v,p) : odd->value(u,v,p);  
  
    }  
private:  
    float invScale;  
    std::shared_ptr<texture> even;  
    std::shared_ptr<texture> odd;  
};
```
![[Checkered.png]]
## Image Textures
We will now use $u,v$ to map our textures. These coordinates define a location on a 2D image. We have our 3D sphere, and we have to somehow map the coordinates of where the ray hit (`p`) to $u,v$.
Texture coordinates for spheres are based on longitude and latitude. We use $(\theta,\phi)$, where $\theta$ is the angle, from -Y to +Y, and $\phi$ is the angle around the Y-axis, which would be  -X to +Z, from +Z to +X to -Z, and looping back to -X.
![[media/theta and phi.png]]
We want $u$ and $v$ to each be in the range $\left[0,1\right]$, so $\phi$ and $\theta$  have to be normalized. To compute $\phi$ and $\theta$ for a given point, we look at the parameterizations:
$$
y=-\cos\theta
$$
$$
x=-\cos(\phi)\sin(\theta)
$$
$$
x=-\sin(\phi)\sin(\theta)
$$
![[media/Sphere Parametrization.png]]
To solve for $\phi$ and $\theta$, we cans simply rearrange. For $\phi$ we can use `std::atan2()`, which returns an angle in the range from $-\pi$ to $\pi$ but they go first from $0$ to $\pi$, and then from $-\pi$ to $0$. To get a range of $0$ to $2\pi$ we can do
$$
atan2(a,b) = atan2(-a,-b) + \pi
$$
thus,
$$
\phi = atan2(-z,x) + \pi
$$
and we can compute $\theta$ rather simply as
$$
\theta = acos(-y)
$$
which gets us this in code:
```cpp
static void getSphere_uv(const sf::Vector3f& p, double& u, double& v) {  
    float theta = std::acos(-p.y);  
    float phi = std::atan2(-p.z,p.x) + pi;  
  
    u = phi/(2*pi);  
    v = theta/pi;  
}
```
Next, we simply just need to get the image data. I will just use `stb_image.h` to load in the image. The most important function here will be `pixelData(int x, int y)`. Given any pixel coordinates, in our case $(u,v)$,  get the color of the pixel.
```cpp
#pragma once  
#define STB_IMAGE_IMPLEMENTATION  
 
  
#include "stb_image.h"  
#include <iostream>  
  
class image {  
public:  
    image () {};  
  
    image(const char* filename) {  
        std::string f = std::string(filename);  
  
        auto imagedir = getenv("PROJECT_ROOT");  
        if (imagedir && load(std::string(imagedir) + "/" + filename)) return;  
        if (load(filename)) return;  
  
        std::cerr << "Could not load image " << f << std::endl;  
    }  
  
    ~image() {  
        STBI_FREE(fdata);  
    }  
  
    bool load(const std::string& filename) {  
        auto n = bytesPerPixel;  
        fdata = stbi_loadf(filename.c_str(),&imageWidth,&imageHeight,&n,bytesPerPixel);  
  
        if (fdata == nullptr) return false;  
        bytesPerScanline = imageWidth * bytesPerPixel;  
        return true;  
    }  
    int width()  const { return (fdata == nullptr) ? 0 : imageWidth; }  
    int height() const { return (fdata == nullptr) ? 0 : imageHeight; }  
  
    const float* pixelData(int x, int y) const {  
        static float magenta[] = {1,0,1};  
        if (fdata == nullptr) return magenta;  
  
        x = std::clamp(x,0,imageWidth-1);  
        y = std::clamp(y,0,imageHeight-1);  
  
        return fdata + y * bytesPerScanline + x * bytesPerPixel;  
  
    }  
private:  
    const int bytesPerPixel = 3;  
    float *fdata = nullptr;  
    int imageWidth = 0;  
    int imageHeight = 0;  
    int bytesPerScanline=0;  
};
```

Now we can type up `value()` for our image texture.
```cpp
sf::Vector3f value(double u, double v, const sf::Vector3f &p) const override {  
    u = interval(0,1).clamp(u);  
    v = 1.0 - interval(0,1).clamp(v);  
  
    auto i = int(u * img.width());  
    auto j = int(v* img.height());  
    auto pixel = img.pixelData(i,j);  
  
    return sf::Vector3f(pixel[0],pixel[1],pixel[2]);  
  
}
```
And now lets test our texture:
```cpp
void earth(hittable_list& world) {  
    cam.samplesPerPixel = 1;  
    cam.maxDepth = 90;  
    cam.fovDegrees = 45.f;  
  
    cam.lookFrom = {2, 2, 1};  
    cam.lookAt = {0, 1, -1};  
  
    auto earthTexture = std::make_shared<imageTexture>("earthTexture.jpg");  
    auto material1 = std::make_shared<lambertian>(earthTexture);  
    world.add(std::make_shared<sphere>(sf::Vector3f(0, 1, -1), 0.4, material1));  
  
}
```
![[Earth.png]]

Now we move on to implementing [[Quadrilaterals]], our next type of primitive.