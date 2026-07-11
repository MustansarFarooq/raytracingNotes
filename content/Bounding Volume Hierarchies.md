A Bounding Volume Hierarchy (BVH) is a structure that stores all the objects in our scene and puts them neatly into a tree. We will approach implementing BVH with axis-aligned bounding boxes (AABB). Essentially, our objects will be put into boxes and we check whether or not our ray intersects this box at all. If it does, then only will we check more in-depth about what objects this ray intersects with within this box. 
We will have to check if our ray intersects the box, defined and bound by axes (plural of axis!). Consider the formula for a ray:
$$
\mathbf{P}(t)=\mathbf{Q}+t\mathbf{D}
$$
![[media/BVH1.png]]
With our ray, we want to see which of these axis lines we intersect. Lets only consider the lines for the x-axis. The point at which our ray's x-component (so considering only ${\mathbf{P}(t)}_x=\mathbf{Q}_x+t_x\mathbf{D}_x$) intersects the first x-axis line, $x_0$, would be 
$$
x_0=\mathbf{Q}_x+t_{x_0}\mathbf{D}_x
$$
Rearranging to solve for $t_{x_0}$ yields:
$$
t_{x_0}=\frac{x_0-\mathbf{Q}_x}{\mathbf{D}_x}
$$
We can do the same to solve for $t_{x_1}$, instead plugging in $x_1$ in place of $x_0$. Once we have the intervals $[t_{x_0},t_{x_1}]$ and $[t_{y_0},t_{y_1}]$, we check if these intervals overlap, and if they do, then they fall within the bounding box. This easily extends to three dimensions, so don't worry!
We do have some issues though. Consider if $\mathbf{D}_x$ is negative. Our interval would look reversed.
![[media/BVH2.png]]
To fix our reversed interval, $[7,3]$, we can easily take the lesser of the two values, $t_{x_0}$ and $t_{x_1}$ to be $t_{x_0}$ and the greater of the two values to be $t_{x_1}$.
$$
t_{x_0}=\mathrm{min}\left(\frac{x_0-\mathbf{Q}_x}{\mathbf{D}_x},\frac{x_1-\mathbf{Q}_x}{\mathbf{D}_x}\right)
$$
$$
t_{x_1}=\mathrm{max}\left(\frac{x_0-\mathbf{Q}_x}{\mathbf{D}_x},\frac{x_1-\mathbf{Q}_x}{\mathbf{D}_x}\right)
$$
Another issue is what if $\mathbf{D}_x=0$? We still do not need to worry, as $t_{x_0}=t_{x_1}$, evaluating both to either $+\infty$ or $-\infty$, if not between $x_0$ or $x_1$. If both $\mathbf{D}_x=0$ an either $x_0-\mathbf{Q}_x=0$ or $x_1-\mathbf{Q}_x=0$, then we will get `NaN`. In that case we will just decide whether it hit the bounding box or not randomly.
In our code, we go on to create the `bvhNode` class that represents our tree. `Hittable` objects are put into these.
The `aabb` class is defined as such:
```cpp
/// @brief Axis Aligned Bounding Box  
class aabb {  
public:  
    interval x, y, z;  
    static const aabb empty, universe;  
  
    aabb(const interval& x, const interval& y, const interval& z) : x(x), y(y), z(z) {}  
    aabb() {};  
    const interval& axis_interval(int n) const {  
        if (n == 1) return y;  
        if (n == 2) return z;  
        return x;  
    }  
  
    aabb(const sf::Vector3f& a, const sf::Vector3f& b)  
    {  
        x = (a.x <= b.x) ? interval(a.x, b.x) : interval(b.x, a.x);  
        y = (a.y <= b.y) ? interval(a.y, b.y) : interval(b.y, a.y);  
        z = (a.z <= b.z) ? interval(a.z, b.z) : interval(b.z, a.z);  
    }  
    aabb(const aabb& box0, const aabb& box1) {  
        x = interval(box0.x, box1.x);  
        y = interval(box0.y, box1.y);  
        z = interval(box0.z, box1.z);  
    }  
  
    int longestAxis() const {  
        if (x.size() > y.size())  
            return x.size() > z.size() ? 0 : 2;  
        else  
            return y.size() > z.size() ? 1 : 2;  
    }  
  
    bool hit(const ray& r, interval ray_t) const {  
        const sf::Vector3f& ray_orig = r.getOrigin();  
        const float* orig = &ray_orig.x;  
        const sf::Vector3f& ray_dir = r.getDirection();  
        const float* dir = &ray_dir.x;  
  
        for (int i = 0; i < 3; i++) {  
            const interval& ax =  axis_interval(i);  
            const double dirInv = 1.f/dir[i];  
  
            float t0 = (ax.min - orig[i]) * dirInv;  
            float t1 = (ax.max - orig[i]) * dirInv;  
  
            //you check for every axis and update ray_t on whether it satisfies the interval ray_t.  
            if (t0 < t1) {  
                if (t0 > ray_t.min) ray_t.min = t0;  
                if (t1 < ray_t.max) ray_t.max = t1;  
            } else {  
                if (t1 > ray_t.min) ray_t.min = t1;  
                if (t0 < ray_t.max) ray_t.max = t0;  
            }  
  
            if (ray_t.max <= ray_t.min) return false;  
        }  
        return true;  
    }  
};  
const aabb aabb::empty    = aabb(interval::empty,    interval::empty,    interval::empty);  
const aabb aabb::universe = aabb(interval::universe, interval::universe, interval::universe);
```
 and the `bvhNode` is defined through the class:
 ```cpp
class bvhNode : public hittable {  
  
public:  
    bvhNode(hittable_list list) : bvhNode(list.objects,0,list.objects.size()) {};  
  
    bvhNode(std::vector<std::shared_ptr<hittable>>& objects, size_t start, size_t end) {  
        bbox = aabb::empty;  
  
        for (size_t object_index = start; object_index < end; object_index++) {  
            bbox = aabb(bbox,objects[object_index]->boundingBox());  
        }  
  
        int axis = bbox.longestAxis();  
        auto comparator = (axis == 0) ? box_x_compare  
                        : (axis == 1) ? box_y_compare  
                        : box_z_compare;  
  
        size_t objectSpan = end-start;  
  
        if (objectSpan == 1 ) {  
            left = right = objects[start];  
        }  
        else if (objectSpan == 2 ) {  
            left = objects[start];  
            right = objects[start+1];  
        }  
        else {  
            std::sort(std::begin(objects) + start, std::begin(objects) + end, comparator);  
  
            auto mid = start + objectSpan/2;  
            left = std::make_shared<bvhNode>(objects, start, mid);  
            right = std::make_shared<bvhNode>(objects, mid, end);  
        }  
    }  
  
    bool hit(const ray &r, interval ray_t, hit_record &rec) const override {  
        if (!bbox.hit(r,ray_t))  
            return false;  
  
        bool hitLeft = left->hit(r,ray_t,rec);  
        bool hitRight = right->hit(r, interval(ray_t.min, hitLeft ? rec.t : ray_t.max),rec);  
        return hitLeft || hitRight;  
  
    }  
  
    aabb boundingBox() const override {return bbox;}  
private:  
    aabb bbox;  
    std::shared_ptr<hittable> left;  
    std::shared_ptr<hittable> right;  
  
    static bool boxCompare( const std::shared_ptr<hittable> a, const std::shared_ptr<hittable> b, int axisIndex) {  
        interval aAxisInterval = a->boundingBox().axis_interval(axisIndex);  
        interval bAxisInterval = b->boundingBox().axis_interval(axisIndex);  
  
        return aAxisInterval.min < bAxisInterval.min;  
    }  
  
    static bool box_x_compare(const std::shared_ptr<hittable> a, const std::shared_ptr<hittable> b) {  
        return boxCompare(a,b,0);  
    }  
    static bool box_y_compare(const std::shared_ptr<hittable> a, const std::shared_ptr<hittable> b) {  
        return boxCompare(a,b,1);  
    }  
    static bool box_z_compare(const std::shared_ptr<hittable> a, const std::shared_ptr<hittable> b) {  
        return boxCompare(a,b,2);  
    }  
};
 ```
 BVH is a hard concept to understand. Don't worry, trace through the code and understand how the recursion works to create a tree. I just really don't want to trace through my own code again.

With BVH: 63.740s
Without BVH: 808.22s

Roughly a 12-14x increase on large scenes.
- [ ] TODO: Explain BVH better.

Now we move on to [[Textures]].