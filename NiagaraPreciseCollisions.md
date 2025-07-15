---
layout: default
title: Niagara precise shaped collisions
---
# Niagara precise shaped collisions

## Updating the pre-existing HLSL for Intra-Particle Collisions to add support for cubes and cylinders.

You can enable precise collisions in Niagara using HLSL and modifying the base PDB intra-particle collisions scratchpad to support it.

## What's supported:
<ul>
  <li>Spherical particles checking against shaped particles for collision</li>
</ul>

## What's not supported:
<ul>
  <li>Shaped particles checking against shaped particles for collision</li>
  <li>Shaped particles checking against spherical particles for collision</li>
</ul>

## Overview

Basically, you make a list of shaped particles that can move other less important particles out of their way. For this support, the shaped particles make use of the existing "unyielding" particle logic -- all non-shaped  particles treat shaped particles as unyielding.<br><br>

All shaped particles should treat each other as spherical and use the regular intra-pdb collisions.<br><br>

You can make tiers of shaped particles.<br><br>

Neighbor Grids should be sized based on particle size. If a grid is too large, you include too many particles and checks get inefficient. If it's too small, a large particle could span across multiple grids and not be detected at its edges for collision.<br><br>

##Example of setting up multiple grids:

<ul>
  <li>Tier 2 (Largest): Use a grid with large extents -- extents equal to double the largest particle's length from center to outermost point.</li>
  <ul>
    <li>These particles will check the Tier 2 grid for regular intra-particle collision.</li>
  </ul>
  <li>Tier 1 (Medium): Use a grid with medium extents</li>
    <ul>
    <li>These particles will check the tier 2 grid for shaped intra-particle collision.</li>
    <li>These particles will check the tier 1 grid for regular intra-particle collision.</li>
  </ul>
  <li>Tier 0 (Small)</li>li>
  <ul>
    <li>These particles will check the tier 2 grid for shaped intra-particle collision.</li>
    <li>These particles will check the tier 1 grid for shaped intra-particle collision.</li>
    <li>These particles will check the tier 0 grid for regular intra-particle collision.</li>
  </ul>
</ul>

For cleanliness and efficiency of collisions, you should check collisions in the following order.
<ol>
  <li>All same tiers against same tiers.</li>
  <ul>
    <li>Tier 1 against Tier 1</li>
    <li>Tier 2 against Tier 2</li>
    <li>Tier 3 against Tier 3</li>
  </ul>
  <li>Small (Tier 3) against Medium (Tier 2)</li>
  <li>Medium (Tier 2) against Large (Tier 1)</li>
</ol>

This lets us get smaller objects to an approximately correct position before the larger objects act on them. The larger particles will then have priority, pushing smaller particles out of the way. This may result in smaller objects overlapping each other, but small objects overlapping is less visible than small objects failing to be pushed by larger objects.<br><br>
If precision matters more than performance you can put all of these in the same update and set it to simulate multiple times. I still suggest running them in this order.

## What variables do we need?
<ol>
  <li>CollisionType (int) -- Box (2), Cylinder (1), or Sphere (0). Sphere collisions could use the regular intra-particle collisions, but for grid size and batching purposes we may as well combine them.</li>
  <li>MaxCollisionRadius (float) -- This is a separate stat from Collision Radius, used for an early out when small objects check larger objects for collision.
  <ul>
    <li>It should be equal to the length to the longest outlying point -- From the center of extents to a corner</li>
    <li>While we could just us CollisionRadius, we will also be using CollisionRadius for intra-particle collisions. It's useful to keep MaxCollisionRadius (early out) separate from CollisionRadius.</li>
    <li>As you may want the collision radius between like-sized particles to be smaller, and we want to use this early-out to eke out all the efficiency we can.</li>
  </ul>
  <li>Extents (Vector) -- Scale * Mesh Extents. Needed for boxes.</li>
  <li>Half Height (float) -- Half height of a cylinder</li>
  <li>CollisionRadius (Float) -- Needed for spheres. Can be calculated by Niagara or manually input. Also used for Cylinder radius.</li>
  <li>Orientation (Quat) -- Needed for boxes and cylinders. Calculated by Niagara.</li>
</ol>

Editing the intra-particle reader is simple.
<ol>
  <li>Copy the base reader scratchpad into another directory and rename it.</li>
  <li>Edit the scratch and add these input pins.</li>
  <li>Edit the HLSL and add this code block<br>
    <details><summary>HLSL</summary><p><script src="https://gist.github.com/michael-royalty/2ea2279b0f605e758b2f58b993052858.js"></script></p></details>
  </li>
</ol>


## Usage:
<ol>
  <li>Create three neighbor grids, all in the same emitter. (Note: If you're using separate emitters you'll want to create the larger neighbor grids under the system instead)</li>
  <li>Populate the three neighbor grids based on the Size variable</li>
  <li>Set up one intra-particle reader (or three in the small emitter, two in the medium, and one in the large if you're using 3 separate emitters. Possibly you can set these up at the system level instead)</li>
  <li>Set up the three intra-particle checks</li>
  <li>Set up the medium to large Shape check</li>
  <li>Set up the small to medium Shape check</li>
</ol>

You may need to limit the max speed of smaller particles, or make your box shapes thicker, if you see particles going into boxes.

# Future Improvements:
## We can make it easier to set up multiple tiers.
<ul>
  <li>Stage 1 (Manual): combine the intra-particle reader to take multiple particle readers and neighbor grids. We can have one intra-particle reader scratch for 2 sizes, and one for 3 sizes.</li>
  <li>Stage 2 (Automatic): combine the multiple readers and neighbor grids into an array. Then we can have as many sizes as we want.</li>
</ul>

## Add variables for efficiency and ease of use.
<ol>
  <li>MaxCollisionRadius (+CollisionType). Used by boxes and cylinders for early out. Used as CollisionRadius for spheres.</li>
  <li>Extents. Vector3 used by boxes.</li>
  <li>HalfHeight and Radius. Vector2 used by cylinders.</li>
  <li>Orientation. Float4 used by boxes and cylinders.</li>
  <li>User Parameter Arrays to store mesh-level CollisionRadius, ShapeType, ShapeHalfHeight, and ShapeExtents. Then we can multiply particle scale by these to get the actual values. Simplifies input into Niagara.</li>
  <li>Mesh (Integer) -- Used to refer to the right element in the User Parameter Arrays</li>
</ol>
