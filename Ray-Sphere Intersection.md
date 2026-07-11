The formula for a sphere is as follows:

$$
x^2 +y^2 + z^2 = r^2
$$

And if we want to center it around a point instead of the origin:

$$
(x-h)^2 + (y-k)^2 + (z-l)^2 = r^2
$$

And it is important to note that:

$$
x-h =-(h-x)
$$
Proven by:
$$
-(h-x)=-h+x = x-h
$$

But since we are squaring $(x-h)$, that means that $-(h-x)$ also gets squared. Since squaring any negative is the same as squaring any positive, we get $(x-h)^2=(h-x)^2$, allowing up to set up: $(x-h)^2 + (y-k)^2 + (z-l)^2 = r^2$ to be equal to $(h-x)^2+(k-y)^2+(l-z)^2=r^2$.

Now *Ray Tracing in One Weekend*, defines a center $\mathbf{C}$ at any such point as described above being $(h,k,l)$, with point $\mathbf{P}$ representing any such point that can satisfy to make a sphere, being $\mathbf{P}=(x,y,z)$. That means that we can rewrite the above equation as:
$$
(\mathbf{C_x}-\mathbf{P_x})^2+(\mathbf{C_y}-\mathbf{P_y})^2+(\mathbf{C_z}-\mathbf{P_z})^2 = r^2
$$
(Recall that $(\mathbf{C}-\mathbf{P})^2 = (\mathbf{P}-\mathbf{C})^2$, so it does not really matter the order.)

Since $(\mathbf{C}-\mathbf{P})$ is a vector in it of itself (You are just subtracting 2 vectors), you can also use the definition of a dot product to also write out the equation of a sphere, by taking the dot product of $(\mathbf{C}-\mathbf{P})$ with itself.

$$
(\mathbf{C}-\mathbf{P}) \cdot (\mathbf{C}-\mathbf{P}) = (\mathbf{C_x}-\mathbf{P_x})^2+(\mathbf{C_y}-\mathbf{P_y})^2+(\mathbf{C_z}-\mathbf{P_z})^2 = r^2
$$

Thus, we could just more simply rewrite the equation of a sphere as:
$$
(\mathbf{C}-\mathbf{P}) \cdot (\mathbf{C}-\mathbf{P}) = r^2
$$
Just remember, $\mathbf{C}$ is the center of the sphere, while $\mathbf{P}$ just represents any point that satisfies that sphere. Knowing that the equation of a ray, $\mathbf{r}(t) = \mathbf{o} + t\,\mathbf{d}$, with origin $\mathbf{o}$, direction $\mathbf{d}$, and a scalar $t$, can get us any value along that ray, we can pass in $\mathbf{r}(t)$ in the place of $\mathbf{P}$, and check if the equation is satisfied. So lets plug it in:
$$
(\mathbf{C}-\mathbf{r}(t)) \cdot (\mathbf{C}-\mathbf{r}(t)) = r^2
$$
Expand $\mathbf{r}(t)$:
$$
(\mathbf{C}-(\mathbf{o}+t\mathbf{d})) \cdot (\mathbf{C}-(\mathbf{o}+t\mathbf{d})) = r^2
$$
Just reorder and expand it a bit more:
$$
(\mathbf{C}-\mathbf{o}-t\mathbf{d})\cdot(\mathbf{C}-\mathbf{o}-t\mathbf{d})=r^2
$$
$$
(-t\mathbf{d}+(\mathbf{C}-\mathbf{o}))\cdot(-t\mathbf{d}+(\mathbf{C}-\mathbf{o}))=r^2
$$
$$
(-t\mathbf{d})\cdot(-t\mathbf{d})+2(-t\mathbf{d})(\mathbf{C}-\mathbf{o})+(\mathbf{C}-\mathbf{o})^2 = r^2
$$
Now lets factor out $t$ wherever we can (**NOTE**: squaring a vector means taking the dot product of itself. $t$ is a scalar, so it can be squared simply, but since $\mathbf{d}$ is the direction vector, it's dot product has to be taken instead. The same applies for when we square $(\mathbf{C}-\mathbf{o})$):
$$
t^2(\mathbf{d}\cdot\mathbf{d})-2t\mathbf{d}\cdot(\mathbf{C}-\mathbf{o}) + (\mathbf{C}-\mathbf{o})\cdot(\mathbf{C}-\mathbf{o})=r^2
$$
$$t^2(\mathbf{d}\cdot\mathbf{d})-2t\mathbf{d}\cdot(\mathbf{C}-\mathbf{o}) + (\mathbf{C}-\mathbf{o})\cdot(\mathbf{C}-\mathbf{o})-r^2=0$$

One neat thing to note in this monstrosity of an equation is that everything is a constant, except for $t$. We know where the center of the circle is, we know the direction of our ray, we know the radius, and we are just solving for where, along the ray, does it intersect the sphere. It may be hard to see, but we have set up a quadratic equation in the form $ax^2+bx+c=0$, allowing us to just use the quadratic formula: 
$$x = \frac{-b \pm \sqrt{b^2 - 4ac}}{2a}$$
where:
$$
a=\mathbf{d}\cdot\mathbf{d}
$$
$$
b=-2\mathbf{d}\cdot(\mathbf{C}-\mathbf{o})
$$
$$
c=(\mathbf{C}-\mathbf{o})\cdot(\mathbf{C}-\mathbf{o})-r^2
$$
A pretty neat note is that depending on the discriminant $b^2 - 4ac$, the number of solutions we can get corresponds with the number of intersections with the sphere. If we have a negative discriminant (i.e. no real solutions), then we get no intersections. If our discriminant is equal to 0, than we get only one solution (and thus one intersection), and if we have a positive discriminant, that means we have 2 solutions and thus 2 intersections (one intersection going into the sphere, and the other going out). 
## Putting it into code
Now lets put in a simple function that returns whether or not we have an intersection with the sphere.

```c++
bool ray::hitSphere(sf::Vector3f sphereCenter, float sphereRadius) {  
    float a = this->direction.dot(this->direction);  
    float b = -2.f * this->direction.dot(sphereCenter-this->origin);  
    float c = (sphereCenter-this->origin).dot((sphereCenter-this->origin))-sphereRadius*sphereRadius;  
    float discriminant = b*b - 4*a*c;  
    return (discriminant>=0);  
}
```

We could additionally also just map out our surface normals, but now we would not want a bool return type, rather a float, so we can get the exact location to where the ray intersects.

```c++
double ray::hitSphere(sf::Vector3f sphereCenter, float sphereRadius) {  
    float a = this->direction.dot(this->direction);  
    float b = -2.f * this->direction.dot(sphereCenter-this->origin);  
    float c = (sphereCenter-this->origin).dot((sphereCenter-this->origin))-sphereRadius*sphereRadius;  
    float discriminant = b*b - 4*a*c;  
    if (discriminant < 0) {  
        return -1;  
    }  
    else {  
        return (-b-std::sqrt(discriminant))/(2.f*a);  
    }  
}  
  
sf::Color ray::rayColor() {  
    float t = hitSphere(sf::Vector3f{0,0,-1},0.5);  
    if (t > 0.f) {  
        sf::Vector3f normal = (this->at(t)-sf::Vector3f{0,0,-1}).normalized();  
        return {  
            static_cast<uint8_t>(127.f*(normal.x+1)),  
            static_cast<uint8_t>(127.f*(normal.y+1)),  
            static_cast<uint8_t>(127.f*(normal.z+1)),  
            255};  
    }  
    
    sf::Vector3f normalizedDir = this->getDirection().normalized();  
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
```
# Simplifying Ray-Sphere Intersection

Lets recall our variables that we are plugging into the quadratic formula:

$$x = \frac{-b \pm \sqrt{b^2 - 4ac}}{2a}$$
where:
$$
a=\mathbf{d}\cdot\mathbf{d}
$$
$$
b=-2\mathbf{d}\cdot(\mathbf{C}-\mathbf{o})
$$
$$
c=(\mathbf{C}-\mathbf{o})\cdot(\mathbf{C}-\mathbf{o})-r^2
$$
Taking the dot product of any vector with itself outputs that vector's magnitude squared. This allows us to set:
$$
a=\|\mathbf{d}\|^{2}
$$
Lets now look at our value for of $b$. Let $h =\mathbf{d}\cdot(\mathbf{C}-\mathbf{o})$. That gives us:
$$
h =\mathbf{d}\cdot(\mathbf{C}-\mathbf{o})
$$
$$
b=-2h
$$
This simplifies the quadratic formula neatly:
$$
x = \frac{-b \pm \sqrt{b^2 - 4ac}}{2a}$$
$$
x=\frac{-(-2h)\pm\sqrt{(-2h)^{2}-4ac}}{2a}
$$
$$
x=\frac{2h\pm\sqrt{4h^2-4ac}}{2a}
$$
$$
x=\frac{2h\pm\sqrt{4(h^2-ac)}}{2a}
$$
$$
x=\frac{h\pm\sqrt{h^2-ac}}{a}
$$
Thus allowing us to simplify our `hitSphere()` function to be simplified and rewritten with a variable for $h$ instead of $b$.
```c++
double ray::hitSphere(sf::Vector3f sphereCenter, float sphereRadius) {  
    float a = this->direction.lengthSquared();  
    float h = this->direction.dot(sphereCenter-this->origin);  
    float c = (sphereCenter-this->origin).dot((sphereCenter-this->origin))-sphereRadius*sphereRadius;  
    float discriminant = h*h - a*c;  
    if (discriminant < 0) {  
        return -1;  
    }  
    else {  
        return (h-std::sqrt(discriminant))/(a);  
    }  
}
```

From here, we, we now want to refactor and clean up our code to help for later. Move on to [[Refactoring and clean up]].