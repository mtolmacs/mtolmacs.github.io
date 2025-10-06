## A New Approach: A\* Pathfinding

Recognizing the limitations of the greedy approach, we turned to a proven solution from the video game world: the **A\* (A-star) pathfinding algorithm**.

### What is A\*?

A\* is a graph traversal and pathfinding algorithm widely used in video games, robotics, and mapping applications. It finds the shortest path between two points by intelligently exploring possible routes, using heuristics to prioritize the most promising paths.

The key insight of A\* is that it balances two factors:

- **g(n)**: The actual cost to reach a node from the start
- **h(n)**: The estimated cost to reach the goal from that node (the heuristic)
- **f(n) = g(n) + h(n)**: The total estimated cost of the path through that node

By always exploring the node with the lowest f(n) value, A\* efficiently finds optimal paths without exhaustively searching every possibility.

### How A\* Works

The algorithm maintains two sets of nodes:

1. **Open set**: Nodes to be evaluated
2. **Closed set**: Nodes already evaluated

The process:

1. Add the start node to the open set
2. Loop until the open set is empty:

   - Select the node with the lowest f(n) score
   - If it's the goal, reconstruct the path and return
   - Move it to the closed set
   - For each neighbor:
     - Calculate tentative g score
     - If the neighbor is in the closed set and the new g score is worse, skip it
     - If the neighbor isn't in the open set or the new g score is better:
       - Update the neighbor's scores
       - Set the current node as the neighbor's parent
       - Add the neighbor to the open set

3. If the open set becomes empty without finding the goal, no path exists

### Binary Heap Optimization

For efficiency, we use a binary heap data structure to optimize node lookup. Instead of linearly searching for the node with the lowest f(n) score, the heap maintains this property automatically, reducing lookup time from O(n) to O(log n).

## Adapting A\* for Elbow Arrows

Implementing A\* for elbow arrows required several domain-specific customizations and optimizations.

### The Non-Uniform Grid Challenge

Operating on a pixel-by-pixel grid would be prohibitively expensive - imagine a 4K canvas with millions of potential nodes to evaluate. Yet we need pixel-precise shape avoidance.

**Solution**: Create a non-uniform grid derived from the shapes themselves.

The algorithm:

1. Collect all shapes that need to be avoided
2. Extract the boundaries (sides) of each shape
3. Extend these boundaries across the entire routing space
4. Where these boundary lines intersect, create potential grid nodes
5. These intersection points become the only valid locations for arrow corner points

This approach provides:

- ✓ Pixel-precise accuracy (nodes align with shape edges)
- ✓ Dramatically reduced search space (hundreds vs. millions of nodes)
- ✓ Natural routing (corners align with shape boundaries)

### Exclusion Zones

To ensure shape avoidance, we implement exclusion zones:

1. For each grid node, check if it falls inside any shape's bounding box
2. If a node is inside a shape to be avoided, mark it as **illegal**
3. The A\* algorithm skips illegal nodes during pathfinding

This simple check ensures that the arrow path never penetrates obstacles, satisfying one of our core requirements.

### Aesthetic Heuristics

While the basic A\* implementation produced better results than the naive approach, visual inspection revealed unintuitive routing in certain edge cases. The paths were optimal in terms of distance but didn't always match human intuition.

To address this, we introduced additional heuristic weights:

#### 1. Bend Penalty

Direction changes are penalized with a linear constant (bendMultiplier). This encourages straighter paths when possible:

- Fewer turns = lower cost
- Routes prefer extending existing segments over creating new ones

#### 2. Backward Prevention

Arrow segments are prohibited from moving "backwards," overlapping with previous segments:

- Prevents ugly loops and backtracking
- Enforces forward progress toward the goal

#### 3. Segment Length Consideration

Longer straight segments are preferred over multiple short segments:

- Weighs segment length in the cost function
- Produces cleaner, more readable arrows

#### 4. Shape Side Awareness

When choosing between routing left or right around an obstacle, the algorithm considers:

- The length of the obstacle's sides
- The relative position of start and end points
- The angle of approach

This helps the arrow choose the more natural route around obstacles.

#### 5. Short Arrow Handling

Special logic for when the start and end points are very close:

- Prevents excessive meandering
- May use a simplified direct route if shapes allow
- Handles overlapping or nearly overlapping shapes gracefully

#### 6. Overlap Management

When connected shapes overlap or are very close together:

- Detects the overlap condition
- Applies special routing rules
- May create a minimal clearance path
- Ensures visual clarity even in crowded diagrams

### Implementation Details

Key components in the codebase:

- **`astar()`** - Core A\* algorithm implementation with elbow arrow constraints
- **`calculateGrid()`** - Generates the non-uniform grid from shape boundaries
- **`generateDynamicAABBs()`** - Creates axis-aligned bounding boxes for shapes
- **`getElbowArrowData()`** - Gathers all necessary data for path calculation
- **`routeElbowArrow()`** - Main entry point that orchestrates the pathfinding
- **`estimateSegmentCount()`** - Heuristic for estimating optimal segment count
- **`normalizeArrowElementUpdate()`** - Converts global path coordinates to element-local coordinates

## Results

The A\* implementation with custom heuristics delivers elbow arrows that:

- ✓ Take optimal or near-optimal routes
- ✓ Minimize the number of segments
- ✓ Avoid all shapes precisely
- ✓ Orient arrow heads correctly
- ✓ Look natural and intuitive
- ✓ Handle edge cases gracefully

## Impact on Users

The hassle-free diagramming enabled by smart elbow arrows has accelerated numerous professional use cases:

- **Faster diagram creation** - No manual arrow routing required
- **Cleaner results** - Professional-looking diagrams without effort
- **Dynamic updates** - Arrows automatically reroute when shapes move
- **Better collaboration** - Teams can quickly iterate on architectural designs
- **Reduced cognitive load** - Focus on content, not on routing mechanics

## Conclusion

What started as a simple feature request - "add elbow arrows" - evolved into a sophisticated pathfinding challenge. By combining classical algorithms (A\*), domain-specific optimizations (non-uniform grids), and carefully tuned heuristics (aesthetic weights), Excalidraw's elbow arrows deliver the intuitive, professional results users expect.

The journey from naive greedy algorithm to sophisticated A\* implementation demonstrates that even seemingly simple UI features can hide deep technical complexity. But when done right, that complexity disappears for the user, leaving only the smooth, natural experience they deserve.
