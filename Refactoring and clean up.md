Next, the book goes into refactoring and setting up some stuff in our ray tracer. I am not going to go too much into the specifics, but just note that now we have a sphere class that inherits from a hittable class.

###### Here are some note-worthy lines:

```cpp
void set_face_normal(const ray& r, const sf::Vector3f& outward_normal) {  
    front_face = r.getDirection().dot(outward_normal) < 0.0;  
    normal = front_face ? outward_normal : -outward_normal;  
}
```

So currently, our normal always faces outwards of the sphere. So if the ray is intersecting through the sphere and goes through it, the ray and the normal would be in the same direction. Because of that, we are unsure of whether we are facing the front side of the sphere. Now we could get around this through this code. Taking the dot product tells us if the ray and normal face the same direction, and storing that in the boolean `front_face`. We then always default the normal to be **against** the ray.
![](media/Normal%20Inversion.png)

Later on, we make a list for hittable objects. Any object that derives from the hittable class, gets put into that list. We have to use `shared_ptr` and pointers in general because when you have a `vector` list of type `hittable`, the inherited class gets sliced off. We cannot store properly a sphere, or later on cubes, and prisms and whatever, because the line
```cpp
std::vector<shared_ptr<hittable>> objects;
```
ends up slicing the inherited class off. So we have to make a list of pointers that point to a single instance of that class. This also helps in making sure we only have one instance of a hittable object in memory. We use `shared_ptr` simply because its flexible and easy to use.

# Camera class
This was generational. It took a while to refactor and figure stuff out, but it eventually boiled down to this:
```cpp
#ifndef SFML_RAYTRACER_CAMERA_H  
#define SFML_RAYTRACER_CAMERA_H  
#include <vector>  
#include <SFML/Graphics/Color.hpp>  
#include "raytracer.h"  
#include "hittable.h"  
  
class camera {  
public:  
  
    camera(const int &w, const int &h) : screenWidth(w), screenHeight(h) { initialize();};  
    void render(const hittable& world, std::vector<std::uint8_t> &image) {  
        for (int y = 0; y < screenHeight; ++y)  
        {  
            for (int x = 0; x < screenWidth; ++x)  
            {  
                int index = (y * screenWidth + x) * 4;  
  
                sf::Color pixelColor{0,0,0,0};  
                 sf::Vector3f pixelPosition = topLeftCorner  
                                             + pixelStepX*(x+0.5f)  //add the 0.5f because we want to get the center of the pixel, not the top left corner of it  
                                             - pixelStepY*(y+0.5f); //subtract because y increases as we go down in sfml  
  
                 sf::Vector3f rayDirection = pixelPosition - cameraPosition; //Ray tracing  
                 ray r = ray(cameraPosition,rayDirection);  
  
                 sf::Color color(rayColor(r, world));  
  
                 setPixel(image, color, index);  
            }  
        }  
    }  
private:  
  
    const int screenHeight;  
    const int screenWidth;  
    //pixel steps  
    sf::Vector3f pixelStepX;  
    sf::Vector3f pixelStepY;  
    sf::Vector3f cameraPosition;  
  
    const float fovDegrees = 90.f;  
    float aspectRatio = static_cast<float>(screenWidth)/screenHeight;  
    sf::Vector3f topLeftCorner;  
  
  
    void setPixel(std::vector<std::uint8_t>& imageBuffer, sf::Color color, int i) {  
        imageBuffer[i] = color.r;  
        imageBuffer[i + 1] = color.g;  
        imageBuffer[i + 2] = color.b;  
        imageBuffer[i + 3] = color.a;  
    }  
  
  
  
  
  
    void initialize() {  
        cameraPosition = {0,0,0};  
        const float viewingPlaneDistance = 1.0f; //the distance between the camera and the center of the viewing plane  
        float viewPlaneWidth = 2 * viewingPlaneDistance * std::tan(degrees_to_radians(fovDegrees)/2);  
        float viewPlaneHeight = viewPlaneWidth/aspectRatio;  
        //distance from left to right, up to down, of each view plane as vec3f  
        sf::Vector3f planeRight = {viewPlaneWidth,0,0};  
        sf::Vector3f planeUp = {0,viewPlaneHeight,0};  
  
        pixelStepX = planeRight/static_cast<float>(screenWidth);  
        pixelStepY = planeUp/static_cast<float>(screenHeight);  
  
        //get the position of the top left corner in coordinate space ig  
        topLeftCorner = cameraPosition + sf::Vector3f{  
            -viewPlaneWidth/2,  
            viewPlaneHeight/2,  
            -viewingPlaneDistance};  
  
    }  
  
    sf::Color rayColor(const ray& r,const hittable& world) const {  
        hit_record rec;  
        if (world.hit(r, interval(0, infinity), rec)) {  
            return {  
                static_cast<uint8_t>(127.f*(rec.normal.x+1)),  
                static_cast<uint8_t>(127.f*(rec.normal.y+1)),  
                static_cast<uint8_t>(127.f*(rec.normal.z+1)),  
                255};  
  
        }  
        sf::Vector3f normalizedDir = r.getDirection().normalized();  
        float a = 0.5f*(normalizedDir.y + 1.0f);  
  
        sf::Vector3f white(255.f, 255.f, 255.f);  
        sf::Vector3f blue(127.f, 178.f, 255.f);  
  
        sf::Vector3f result = (1.0f-a)*white + a*blue; //a nice sky-blue color  
  
        return {  
            static_cast<uint8_t>(result.x),  
            static_cast<uint8_t>(result.y),  
            static_cast<uint8_t>(result.z),  
            255};  
    }  
};  
  
#endif //SFML_RAYTRACER_CAMERA_H
```
*`camera.h`*

**NOTE:** Later we switch from color to always being a `Vector3f` with values on a scale from 0.0-1.0, which later get translated on to SFML's `uint8_t` scale only in the render function.

Now lets move on to actually making our spheres with [materials](Materials.md).