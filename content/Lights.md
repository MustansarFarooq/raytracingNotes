Previously, our `rayColor()` function only considered **reflected light**. If a ray hit an object, the material's `scatter()` function generated a new ray and an attenuation (the fraction of light that the surface reflects). The returned color was simply the reflected light from the next bounce:
```cpp
if (world.hit(r, interval(0.001, infinity), rec)) {
            ray scattered;
            sf::Vector3f attenuation;
            if(rec.mat->scatter(r,rec,attenuation, scattered)) {
                return attenuation*rayColor(scattered,depth-1,world);
            }
            return sf::Vector3f(0,0,0);

        }
```
If we hit an object, we recursively set attenuation (the color that we retain from the object). With lights though, we just add our emitted light.If the material does not scatter, the ray stops and returns the emitted color. Otherwise, the material may both emit light and reflect incoming light.
```cpp
sf::Vector3f rayColor(const ray &r, int depth, const hittable &world) const {  
    hit_record rec;  
  
    if (depth <= 0) return {0,0,0};  
  
    if (!world.hit(r, interval(0.001, infinity), rec))  
        return background;  
  
    ray scattered;  
    sf::Vector3f attenuation;  
    sf::Vector3f emmitted = rec.mat->emmited(rec.u,rec.v,rec.p);  
    if(!rec.mat->scatter(r,rec,attenuation, scattered)) {  
        return emmitted;  
    }  
    return emmitted + (attenuation * rayColor(scattered, depth-1, world));  
  
}
```
