We will introduce a new primitive, the `quad`, which will technically be a parallelogram. To define this quad, we will have $\mathbf{Q}$, the starting corner, and vectors $\mathbf{u}$ and $\mathbf{v}$, each representing a side.
![[media/Quad.png]]
## Ray-Plane Intersection
Our approach to checking for an intersection with a quadrilateral is to first see if our ray intersects the plane on which the quadrilateral lies upon, and then to check if our ray is in the section of the plane in which the interior of the quadrilateral lies. Recall the formula for a plane: 
$$
Ax+By+Cz = D
$$
Where $(A,B,C)$ are the normal for that plane (which we will represent with $\mathbf{n}$), and $(x,y,z)$ (which we will represent with the vector, $\mathbf{v}$) are points that satisfy the equation, lying on the plane, and $D$ is a constant. Therefore we can rewrite the formula for a plane as:
$$
\mathbf{n}\cdot\mathbf{v} = D
$$
Now we can just plug in the formula for a ray, $\mathbf{r}(t) = \mathbf{o} + t\,\mathbf{d}$, into $\mathbf{v}$.
$$
\mathbf{n}\cdot \left(\mathbf{o} + t\,\mathbf{d}\right) = D
$$
$$
\mathbf{n}\cdot \mathbf{o} + \mathbf{n}\cdot t\mathbf{d}=D
$$
$$
\mathbf{n}\cdot \mathbf{o}+t\left(\mathbf{n}\cdot \mathbf{d}\right)=D
$$
$$
t=\frac{D-\mathbf{n}\cdot \mathbf{o}}{\mathbf{n}\cdot \mathbf{d}}
$$
Okay, so far we now have where on a plane our ray has hit. However, we don't actually have the equation for this plane. Recall that we need $\mathbf{n}$, which is the normal of the plane. Well the normal is simply the cross product of the $\mathbf{u}$ and $\mathbf{v}$, normalized.
$$
\mathbf{n}=\frac{\mathbf{u}\times \mathbf{v}}{\left\| \mathbf{u\times v} \right\|}
$$
Finally, to complete our equation for a plane, we need to solve for D. Given a point $(x,y,z)$ that lies on our quad, we can solve for D. We know that $\mathbf{Q}$ lies in our quad on this plane, so we can plug it in.
$$
\mathbf{n}\cdot \mathbf{v}=D
$$
$$
\mathbf{n}\cdot \mathbf{Q}=D
$$
## The Planar Coordinates
Now we have the plane, and we know the point on which our plane lies on. We now want to check if on this plane, our point lies within our quadrilateral. For this task, we need to orient our points on this plane. In other words, we need a way to position each point on this plane in our own coordinate system. To do this, we can express each point on the plane, where given any point $\mathbf{P}$, we can use two scalar values $\alpha$ and $\beta$:
$$
\mathbf{P} = \mathbf{Q} +\alpha\mathbf{u} + \beta \mathbf{v}
$$
Well, how do we find $\alpha$ and $\beta$? Consider that $\mathbf{p}$ is the vector from $\mathbf{Q}$ to $\mathbf{P}$. Remember that $\mathbf{P}$ is the point where our ray hits, while $\mathbf{p}$ is the vector from $\mathbf{Q}$ to $\mathbf{P}$.
$$
\mathbf{p}=\mathbf{P}-\mathbf{Q}=\alpha \mathbf{u}+\beta \mathbf{v}
$$
Now to solve for $\alpha$ and $\beta$, cross the equation for $\mathbf{p}$ with $\mathbf{u}$ and $\mathbf{v}$.
$$
\mathbf{u\times p}=\mathbf{u}\times(\alpha \mathbf{u}+\beta \mathbf{v})=\alpha(\mathbf{u\times u})+\beta(\mathbf{u\times v})
$$
$$
\mathbf{v\times p}=\mathbf{v}\times(\alpha \mathbf{u}+\beta \mathbf{v})=\alpha(\mathbf{v\times u})+\beta(\mathbf{v\times v})
$$
Any vector crossed with itself is just zero, so we can further simplify to
$$
\mathbf{u\times p}=\beta(\mathbf{u\times v})
$$
$$
\mathbf{v\times p}=\alpha(\mathbf{v\times u})
$$
You cannot divide by vectors. You can divide by scalars however. We can get a scalar by taking the dot product of both sides with the normal.
$$
\mathbf{n}\cdot(\mathbf{u}\times \mathbf{p}) = \mathbf{n}\cdot \beta(\mathbf{u\times v})
$$
$$
\mathbf{n}\cdot(\mathbf{v}\times \mathbf{p}) = \mathbf{n}\cdot \alpha(\mathbf{v\times u})
$$
And so:
$$
\alpha=\frac{\mathbf{n}\cdot(\mathbf{v\times p})}{\mathbf{n\cdot}(\mathbf{v\times u})}
$$
$$
\beta=\frac{\mathbf{n}\cdot(\mathbf{u\times p})}{\mathbf{n\cdot}(\mathbf{u\times v})}
$$
Since $(\mathbf{a}\times \mathbf{b})=-(\mathbf{b}\times \mathbf{a})$, we can reverse the order of the cross product in **both** the numerator and denominator of $\alpha$.

$$
\alpha=\frac{\mathbf{n}\cdot(\mathbf{p\times v})}{\mathbf{n\cdot}(\mathbf{u\times v})}
$$
$$
\beta=\frac{\mathbf{n}\cdot(\mathbf{u\times p})}{\mathbf{n\cdot}(\mathbf{u\times v})}
$$
Well, both $\alpha$ and $\beta$ have $\frac{\mathbf{n}}{\mathbf{n\cdot}(\mathbf{u\times v})}$ in common now. Notice how this value is always fixed for every quad we make. Additionally, $\mathbf{u\times v}=\mathbf{n}$. This value, we can cache and store as a variable, $\mathbf{w}$.
$$
\mathbf{w=}\frac{\mathbf{n}}{\mathbf{n\cdot n}}
$$
$$
\alpha=\mathbf{w\cdot}(\mathbf{p\times v})
$$
$$
\beta=\mathbf{w\cdot}(\mathbf{u\times p})
$$
## Interior Testing
We now have where on our plane our ray hit. If a point with coordinates $(\alpha,\beta)$ lie on our quadrilateral, both $\alpha$ and $\beta$ lie in the intervals $[0,1]$.
## The `quad` class
To put this all into code:
```cpp
class quad : public hittable {  
public:  
  
  
    quad(const sf::Vector3f& q,const sf::Vector3f& u,const sf::Vector3f& v, std::shared_ptr<material> m) : mat(m), q(q), u(u), v(v) {  
        sf::Vector3f n = u.cross(v);  
        normal = n.normalized();  
        D = normal.dot(q);  
        w = n / (n.dot(n));  
        setBoundingBox();  
    }  
    void setBoundingBox() {  
        aabb quadDiagonal1 = aabb(q,q+u+v);  
        aabb quadDiagonal2 = aabb(q+u,q+v);  
        bbox = aabb(quadDiagonal1,quadDiagonal2);  
    }  
  
    aabb boundingBox() const override {return bbox;}  
  
  
  
    bool hit(const ray &r, interval ray_t, hit_record &rec) const override {  
        auto denom = normal.dot(r.getDirection());  
        if (std::fabs(denom) < 1e-8) return false;  
  
        auto t = (D-(normal.dot(r.getOrigin())))/denom;  
        if (!ray_t.contains(t)) return false;  
  
        sf::Vector3f intersection = r.at(t);  
        sf::Vector3f planeHitPointVector = intersection - q;  
        float alpha = w.dot(planeHitPointVector.cross(v));  
        float beta = w.dot(u.cross(planeHitPointVector));  
  
        if (!inInterior(alpha, beta, rec)) return false;  
  
  
        rec.t = t;  
        rec.p = intersection;  
        rec.mat = mat;  
        rec.set_face_normal(r,normal);  
        return true;  
    };  
  
    virtual bool inInterior(float a, float b, hit_record & rec) const {  
        interval unitInterval = interval(0,1);  
        if (!unitInterval.contains(a) || !unitInterval.contains(b)) return false;  
        rec.u = a;  
        rec.v = b;  
  
        return true;  
    }  
  
private:  
    std::shared_ptr<material> mat;  
    sf::Vector3f q,u,v;  
    sf::Vector3f normal;  
    sf::Vector3f w;  
    aabb bbox;  
    float D;  
};
```


![[Quads.png]]