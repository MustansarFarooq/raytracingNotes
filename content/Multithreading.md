Here comes a pretty large optimization. With multithreading, we can have our main render loop being run on multiple threads instead of one. Currently, our render loop is rather sequential.
```cpp
        for (int y = 0; y < screenHeight; ++y) {
            for (int x = 0; x < screenWidth; ++x) {
				...
                for (int sample = 0; sample < samplesPerPixel; ++sample) {
                    ray r = getRay(x, y);
                    sf::Vector3f c = rayColor(r, maxDepth, world);
                    pixelColor += sf::Vector3f(c.x, c.y, c.z);
                }
                ...
                setPixel(backBuffer, finalColor, index*4);
                ++completedPixels;
            }
        }
```
We go left to right, up to down, pixel by pixel, computing each ray once per pixel. We could shift this work to multiple threads, so that at any single moment, we have multiple threads working on multiple pixels. To accomplish this, lets first group our pixels. We are going to put our pixels into groups. If we just subdivided each section of the screen, parts like the sky would just quickly render, and that thread would then be done with its work and go away. Instead, lets make the screen into tiles, and assign a new tile for a thread to work on after it is done with its tile. We start with making a `tile` struct:
```cpp
struct tile {  
    int x0, y0, x1, y1;  
};
```
`x0`-`x1` are the coordinates for the span of the tile along the width of our screen. Likewise for `y0` and `y1`. We make, `std::vector<tile> tiles;` which contains all our tiles, and we populate it in our constructor:
```cpp
for (int y = 0; y < screenHeight; y += TILE_SIZE) {  
    for (int x = 0; x < screenWidth; x += TILE_SIZE) {  
        tiles.push_back({  
            x,  
            y,  
            std::min(x + TILE_SIZE, screenWidth),  
            std::min(y + TILE_SIZE, screenHeight)  
        });  
    }  
}
```
`TILE_SIZE` is a `const` I have, that just defines how large a tile should be. Now we have our tiles, lets shift our render loop into a `renderTile()` function.
```cpp
void renderTile(const tile& tile, const hittable& world) {  
        for (int y = tile.y0; y< tile.y1; y++) {  
            for (int x = tile.x0; x < tile.x1; x++) {  
  
                int index = y*screenWidth+x;  
                sf::Vector3f pixelColor{0, 0, 0};  
  
                for (int sample = 0; sample < samplesPerPixel; ++sample) {  
                    ray r = getRay(x, y);  
                    sf::Vector3f c = rayColor(r, maxDepth, world);  
                    pixelColor += sf::Vector3f(c.x, c.y, c.z);  
                }  
                accumulated[index] += pixelColor;  
                pixelColor = accumulated[index] / accumulatedFrame;  
                pixelColor *= static_cast<float>(pixelSamplesScale);  
                pixelColor.x = std::sqrt(pixelColor.x);  
                pixelColor.y = std::sqrt(pixelColor.y);  
                pixelColor.z = std::sqrt(pixelColor.z);  
                sf::Color finalColor(  
                    static_cast<uint8_t>(255.999f * pixelColor.x),  
                    static_cast<uint8_t>(255.999f * pixelColor.y),  
                    static_cast<uint8_t>(255.999f * pixelColor.z),  
                    255  
                );  
  
                setPixel(backBuffer, finalColor, index * 4);  
  
            }  
        }  
        completedPixels += (TILE_SIZE*TILE_SIZE);  
    }  
};
```
Nothing has really changed so far. We have a separate function now for rendering each tile. Our each individual thread is going to be calling upon this function to render whatever tile they were assigned. Lets move on to actually writing out how our tiles will be assigned to each thread. 
We will have an atomic counter, called `nextTile`, which will give us the index of the tile we have rendered so far. For the duration of our thread, we:
1. Get our index for the next tile to render
2. Get the tile given our index.
3. Call `renderTile()`, passing in our tile and world.
```cpp
nextTile = 0;  
auto worker = [&]() {  
  
        while (true) {  
            int tileIndex = nextTile.fetch_add(1);  
  
            if (tileIndex>=tiles.size())  
                break;  
  
            const tile& tile = tiles[tileIndex];  
  
            renderTile(tile,world);  
        }  
};
```
And now, for every thread lets assign it:
```cpp
unsigned threadCount = std::thread::hardware_concurrency();  
{  
    std::vector<std::jthread> workers;  
    for (int i = 0; i < threadCount; ++i) {  
        workers.emplace_back(worker);  
    }  
}
```
And there we have it. Multithreading isn't too hard. Our performance gains are pretty large.