## 3D-Space-Lander

A solo final project for `CS134 - Computer Game Design and Programming` at _San Jose State University_. This project is a 3-D game where a player
controls a space lander in a Moon-like environment.

## Languages/Technologies
* C++
* openFrameworks

## Background and Overview

### Collision Detection

The collision detection is handled through a data structure called an **Octree**. An Octree is a data structure where each node have
exactly eight children. In a 3-D environment, Octrees are visualized through boxes, where each box represent a node in the tree.

![octree](https://i.imgur.com/uYchObr.png)



**Here is how the Octree works:** Assume we have a mesh with a list of 3D points which can be indexed by i. Starting condition is that the entire mesh is in a bounding
box (the smallest box containing all points). We then subdivide the box into 8 equally sized boxes. Each box represents a node and the node
will contain the box's information, list of point indices, and list of child boxes. Continue subdividing the newly created boxes until
it reaches the smallest box possible; this box will contain one point. Here is a code snippet containing the subdivide function:

`Octree.cpp`
```c++
void Octree::subdivide(const ofMesh & mesh, TreeNode & node, int numLevels, int level) {
	if (node.points.size() > 2) { //a condition to stop recursion if a specified level is reached
		vector<Box> temp1;
		TreeNode temp2;
		subDivideBox8(node.box, temp1); //first subdivide the box into 8 smaller boxes
		for (int i = 0; i < temp1.size(); i++) {
			temp2.box = temp1[i];

			getMeshPointsInBox(mesh, node.points, temp2.box, temp2.points); //checks if children's points correspond to parent's points

			if (temp2.points.size() > 0) //a child node will become the child node of the parent if there's corresponding points
				node.children.push_back(temp2);
		}

		node.points.clear();
		level++;
		for (int i = 0; i < node.children.size(); i++) {
			if (node.children[i].points.size() != 1) { //call recursive function as long as the child's number of points is not 1
				node.children[i].intersects = true;
				subdivide(mesh, node.children[i], numLevels, level);
			}
		}
	}
}
```

### Physics and Particle Simulations

Physics and particles are applied to the space lander. Whenever a player manuevers the lander, force is applied to the direction (a.k.a. velocity) 
to give it a realistic feeling. When the lander is idle, a **drift force** is applied. The drift force allows the lander to slightly move in random directions. 
Multiple forces are applied to the lander, such as **turbulence force** and **cyclic force**. 

Particles are also applied to the lander by attaching a **Particle Emitter** on the lander.
Particles are contained and managed in a system and are added, updated, and deleted when necessary. The Particle Emitter is an object that is responsible for spawning new particles.
In the context of the the game, the particle emitter is the _engine_ of the lander. Whenever a player press the spacebar, particles will
spawn underneath the lander in a circular fashion and a **thruster force** is applied to the lander to increase acceleration the longer
the player holds on the spacebar. 

The physics and particle simulations were written from scratch. Here are some code snippets of these simulations in action:

`ParticleSystem.h`

```c++
class ParticleForce {
protected:
public:
	bool applyOnce = false;
	bool applied = false;
	MoveDir direction = noDirection;
	virtual void updateForce(Particle *) = 0;
};
```

```c++
class TurbulenceForce : public ParticleForce {
	ofVec3f tmin, tmax;
public:
	void set(const ofVec3f &min, const ofVec3f &max) { tmin = min; tmax = max; }
	TurbulenceForce(const ofVec3f & min, const ofVec3f &max);
	TurbulenceForce() { tmin.set(0, 0, 0); tmax.set(0, 0, 0); }
	void updateForce(Particle *);
};
```

`ParticleSystem.cpp`

```c++
TurbulenceForce::TurbulenceForce(const ofVec3f &min, const ofVec3f &max) {
	tmin = min;
	tmax = max;
}

void TurbulenceForce::updateForce(Particle * particle) {
	//
	// We are going to add a little "noise" to a particles
	// forces to achieve a more natual look to the motion
	//
	particle->forces.x += ofRandom(tmin.x, tmax.x);
	particle->forces.y += ofRandom(tmin.y, tmax.y);
	particle->forces.z += ofRandom(tmin.z, tmax.z);
}
```

The following code snippet updates all particles' forces contained in the vector container. Whenever a new force is added to a particle,
this algorithm will update it accordingly.

```c++
vector<Particle> particles;
vector<ParticleForce *> forces;
```

```c++
// update forces on all particles first 
//
for (int i = 0; i < particles.size(); i++) {
  for (int k = 0; k < forces.size(); k++) {
    if (!forces[k]->applied) {
      forces[k]->direction = newDirection;
      forces[k]->updateForce(&particles[i]);
    }
  }
}

// update all forces only applied once to "applied"
// so they are not applied again.
//
for (int i = 0; i < forces.size(); i++) {
  if (forces[i]->applyOnce)
    forces[i]->applied = true;
}
````


### Camera Functionalities

The game supports the ability to switch between multiple camera views. The player can switch to a "tracking" camera that stays
aimed at the spacecraft from a fixed location. They can also switch to two onboard cameras: one for a side view overlooking the surface 
and another for a view looking directly down at the surface.

In addition, the player have access to the EasyCam camera. They are able to navigate anywhere they like over the surface. The camera can
also retarget the view to a point or the lander.

