# Week 02 — 3D Pac-Man Style Maze Game

Build a 3D top-down maze game where the player navigates randomly generated corridors collecting dots while being chased by an enemy. The maze is different every time you play.

---

## What you need

- Unity 2022+ with the **3D (Built-In Render Pipeline)** template
- A new project named `Walker_Pacman`

---

## Project setup

### Camera
1. Select **Main Camera** in the Hierarchy
2. Set Position: `(0, 20, 0)`, Rotation: `(90, 0, 0)`
3. Set **Projection** to `Orthographic`

### Tags
Go to **Edit → Project Settings → Tags and Layers** and add these tags:
- `Player`
- `Enemy`
- `Dot`

---

## Step 1 — Create your four prefabs

The maze generator script will spawn everything at runtime, so you need four prefabs ready before writing any code.

### Wall prefab
1. Right-click Hierarchy → **3D Object → Cube** → rename it `Wall`
2. Drag it from the Hierarchy into the Project panel to save as a prefab
3. Delete it from the scene

### Dot prefab
1. Right-click Hierarchy → **3D Object → Sphere** → rename it `Dot`
2. Scale it to `(0.2, 0.2, 0.2)`
3. Add a **SphereCollider** → check **Is Trigger**
4. Tag it `Dot`
5. Drag to Project panel → delete from scene

### Player prefab
1. Right-click Hierarchy → **3D Object → Sphere** → rename it `Player`
2. Add **Rigidbody** → expand **Constraints** → check **Freeze Rotation X, Y, Z**
3. Tag it `Player`
4. Drag to Project panel → delete from scene

### Enemy prefab
1. Right-click Hierarchy → **3D Object → Sphere** → rename it `Enemy`
2. Add **Rigidbody** → expand **Constraints** → check **Freeze Rotation X, Y, Z**
3. Tag it `Enemy`
4. Drag to Project panel → delete from scene

---

## Step 2 — Maze generator

This script uses Prim's algorithm to generate a random maze each time you play. It also places all dots, the player, and the enemy automatically.

Create a C# script named `MazeGenerator`. Create an empty GameObject in the scene, name it `MazeGenerator`, and attach the script to it.

```csharp
using System.Collections.Generic;
using UnityEngine;

public class MazeGenerator : MonoBehaviour
{
    [Header("Maze Settings")]
    public int width = 21;
    public int height = 21;
    public float cellSize = 2f;

    [Header("Prefabs")]
    public GameObject wallPrefab;
    public GameObject dotPrefab;
    public GameObject playerPrefab;
    public GameObject enemyPrefab;

    private bool[,] maze;

    void Start()
    {
        maze = new bool[width, height];
        GenerateMaze();
        BuildMaze();
        PlaceDots();
        PlacePlayer();
        PlaceEnemy();
    }

    void GenerateMaze()
    {
        for (int x = 0; x < width; x++)
            for (int y = 0; y < height; y++)
                maze[x, y] = false;

        maze[1, 1] = true;
        List<Vector2Int> frontier = new List<Vector2Int>();
        AddFrontier(1, 1, frontier);

        while (frontier.Count > 0)
        {
            int index = Random.Range(0, frontier.Count);
            Vector2Int cell = frontier[index];
            frontier.RemoveAt(index);

            List<Vector2Int> neighbors = GetPassageNeighbors(cell.x, cell.y);
            if (neighbors.Count > 0)
            {
                Vector2Int neighbor = neighbors[Random.Range(0, neighbors.Count)];
                int mx = (cell.x + neighbor.x) / 2;
                int my = (cell.y + neighbor.y) / 2;
                maze[cell.x, cell.y] = true;
                maze[mx, my] = true;
                AddFrontier(cell.x, cell.y, frontier);
            }
        }
    }

    void AddFrontier(int x, int y, List<Vector2Int> frontier)
    {
        int[] dx = { 0, 0, 2, -2 };
        int[] dy = { 2, -2, 0, 0 };
        for (int i = 0; i < 4; i++)
        {
            int nx = x + dx[i];
            int ny = y + dy[i];
            if (nx > 0 && nx < width - 1 && ny > 0 && ny < height - 1 && !maze[nx, ny])
            {
                Vector2Int candidate = new Vector2Int(nx, ny);
                if (!frontier.Contains(candidate))
                    frontier.Add(candidate);
            }
        }
    }

    List<Vector2Int> GetPassageNeighbors(int x, int y)
    {
        List<Vector2Int> result = new List<Vector2Int>();
        int[] dx = { 0, 0, 2, -2 };
        int[] dy = { 2, -2, 0, 0 };
        for (int i = 0; i < 4; i++)
        {
            int nx = x + dx[i];
            int ny = y + dy[i];
            if (nx > 0 && nx < width - 1 && ny > 0 && ny < height - 1 && maze[nx, ny])
                result.Add(new Vector2Int(nx, ny));
        }
        return result;
    }

    void BuildMaze()
    {
        for (int x = 0; x < width; x++)
        {
            for (int y = 0; y < height; y++)
            {
                if (!maze[x, y])
                {
                    Vector3 pos = new Vector3(x * cellSize, 0, y * cellSize);
                    GameObject wall = Instantiate(wallPrefab, pos, Quaternion.identity);
                    wall.transform.localScale = new Vector3(cellSize, cellSize, cellSize);
                    wall.transform.parent = transform;
                }
            }
        }
    }

    void PlaceDots()
    {
        for (int x = 0; x < width; x++)
        {
            for (int y = 0; y < height; y++)
            {
                if (maze[x, y])
                {
                    Vector3 pos = new Vector3(x * cellSize, 0.5f, y * cellSize);
                    Instantiate(dotPrefab, pos, Quaternion.identity);
                }
            }
        }
    }

    void PlacePlayer()
    {
        Vector3 pos = new Vector3(1 * cellSize, 0.5f, 1 * cellSize);
        Instantiate(playerPrefab, pos, Quaternion.identity);
    }

    void PlaceEnemy()
    {
        Vector3 pos = new Vector3((width - 2) * cellSize, 0.5f, (height - 2) * cellSize);
        Instantiate(enemyPrefab, pos, Quaternion.identity);
    }
}
```

**After attaching the script**, select the `MazeGenerator` GameObject and drag your four prefabs into the Inspector slots:
- `Wall Prefab` → Wall
- `Dot Prefab` → Dot
- `Player Prefab` → Player
- `Enemy Prefab` → Enemy

**Solid when:** press Play and a maze generates with walls, dots, a player sphere, and an enemy sphere visible from above.

---

## Step 3 — Player movement

Create a C# script named `PlayerMovement` and attach it to the **Player prefab** (double-click the prefab to open it, add the script, save):

```csharp
using UnityEngine;

public class PlayerMovement : MonoBehaviour
{
    public float moveSpeed = 5f;

    private Rigidbody rb;

    void Awake()
    {
        rb = GetComponent<Rigidbody>();
    }

    void FixedUpdate()
    {
        float h = Input.GetAxisRaw("Horizontal");
        float v = Input.GetAxisRaw("Vertical");
        Vector3 direction = new Vector3(h, 0, v).normalized;
        rb.linearVelocity = direction * moveSpeed;
    }
}
```

**Solid when:** player sphere moves through the maze corridors with WASD, stops cleanly on key release, and cannot walk through walls. Recommended `moveSpeed = 5`.

---

## Step 4 — Dot collection

Create a C# script named `DotCollector` and attach it to the **Player prefab**:

```csharp
using UnityEngine;

public class DotCollector : MonoBehaviour
{
    public int dotsCollected = 0;

    void OnTriggerEnter(Collider other)
    {
        if (other.CompareTag("Dot"))
        {
            dotsCollected++;
            Destroy(other.gameObject);
            Debug.Log("Dots collected: " + dotsCollected);
        }
    }
}
```

**Solid when:** walking over a dot destroys it and the collected count increments in the Console. No win condition yet.

---

## Step 5 — Enemy chase

Create a C# script named `EnemyChase` and attach it to the **Enemy prefab**:

```csharp
using UnityEngine;

public class EnemyChase : MonoBehaviour
{
    public float chaseSpeed = 2.25f;

    private Transform player;
    private Rigidbody rb;

    void Start()
    {
        rb = GetComponent<Rigidbody>();
        GameObject p = GameObject.FindWithTag("Player");
        if (p != null) player = p.transform;
    }

    void FixedUpdate()
    {
        if (player == null) return;
        Vector3 direction = (player.position - transform.position).normalized;
        direction.y = 0;
        rb.linearVelocity = direction * chaseSpeed;
    }
}
```

**Solid when:** enemy moves toward the player, is beatable but threatening, and doesn't clip through walls perfectly (full pathfinding is out of scope for this week — the enemy will push against walls but still feel threatening in open corridors). Recommended `chaseSpeed = 2.25`.

---

## Step 6 — Win condition

Create a C# script named `WinCondition` and attach it to the **MazeGenerator** GameObject:

```csharp
using UnityEngine;

public class WinCondition : MonoBehaviour
{
    private int totalDots = 0;
    private DotCollector collector;

    void Start()
    {
        totalDots = FindObjectsByType<SphereCollider>(FindObjectsSortMode.None).Length;
        collector = FindFirstObjectByType<DotCollector>();
    }

    void Update()
    {
        if (collector == null) return;
        if (collector.dotsCollected >= totalDots)
        {
            Debug.Log("You Win!");
            enabled = false;
        }
    }
}
```

**Solid when:** collecting all dots prints "You Win!" to the Console.

---

## Locked values

| Setting | Value |
|---|---|
| moveSpeed | 5 |
| chaseSpeed | 2.25 |
| Maze size | 21x21 |
| Cell size | 2 |

---

## Common issues

| Problem | Fix |
|---|---|
| Maze doesn't generate | Make sure all four prefabs are assigned in the MazeGenerator Inspector slots |
| Player falls through the floor | Add a floor plane — right-click Hierarchy → 3D Object → Plane, scale it to cover the maze |
| Player floats above maze | Lower the camera or adjust the player Y spawn position in MazeGenerator |
| Dots not disappearing | Check Dot prefab has Is Trigger checked on its SphereCollider and is tagged `Dot` |
| Enemy phases through walls | Expected for week 2 — full NavMesh pathfinding is a future addition |
| Camera doesn't see full maze | Increase the camera's Orthographic Size to around 25-30 |
