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
2. Set Position: `(20, 30, 20)`, Rotation: `(90, 0, 0)`
3. Set **Projection** to `Orthographic`
4. Set **Orthographic Size** to `25`
5. Set **Background** color to black `(0, 0, 0)`

### Tags
Go to **Edit → Project Settings → Tags and Layers** and add these tags:
- `Player`
- `Enemy`
- `Dot`

### Folder structure
Create these folders in the Project panel. Right-click → **Create → Folder**:

```
Assets/
├── Prefabs/
├── Scripts/
├── Materials/
└── Scenes/
```

---

## Step 1 — Create materials

Right-click **Materials/** → **Create → Material** — make 5 materials:

| Material name | Albedo color | Used on |
|---|---|---|
| `WallMat` | Dark blue `(0, 50, 150)` | Wall prefab |
| `DotMat` | Bright yellow `(255, 220, 0)` | Dot prefab |
| `PlayerMat` | Bright green `(0, 200, 80)` | Player prefab |
| `EnemyMat` | Bright red `(200, 0, 0)` | Enemy prefab |
| `FloorMat` | Dark grey `(40, 40, 40)` | Floor plane |

---

## Step 2 — Create prefabs

Save all prefabs into `Prefabs/`. For each one: create it, set it up, drag the matching material onto it, drag it into the Prefabs folder, then delete it from the scene.

### Wall prefab
1. Right-click Hierarchy → **3D Object → Cube** → rename `Wall`
2. Drag `WallMat` onto it
3. Save as prefab → delete from scene

### Dot prefab
1. Right-click Hierarchy → **3D Object → Sphere** → rename `Dot`
2. Scale: `(0.3, 0.3, 0.3)`
3. Add **SphereCollider** → check **Is Trigger** → set Radius to `0.8`
4. Tag: `Dot`
5. Drag `DotMat` onto it
6. Save as prefab → delete from scene

### Player prefab
1. Right-click Hierarchy → **3D Object → Sphere** → rename `Player`
2. Add **Rigidbody** → uncheck **Use Gravity** → expand **Constraints** → check **Freeze Position Y** and **Freeze Rotation X, Y, Z**
3. Tag: `Player`
4. Drag `PlayerMat` onto it
5. Save as prefab → delete from scene

### Enemy prefab
1. Right-click Hierarchy → **3D Object → Sphere** → rename `Enemy`
2. Add **Rigidbody** → uncheck **Use Gravity** → expand **Constraints** → check **Freeze Position Y** and **Freeze Rotation X, Y, Z**
3. Tag: `Enemy`
4. Drag `EnemyMat` onto it
5. Save as prefab → delete from scene

---

## Step 3 — Physics material

Create a physics material so the player and enemy slide along walls instead of sticking:

1. Right-click **Materials/** → **Create → Physics Material** → name it `SlipperyMat`
2. Set **Dynamic Friction**: `0`, **Static Friction**: `0`, **Bounciness**: `0`, **Friction Combine**: `Minimum`
3. Open the **Player prefab** → click its **SphereCollider** → drag `SlipperyMat` into the **Material** slot
4. Do the same for the **Enemy prefab** and the **Wall prefab**

---

## Step 4 — Floor

1. Right-click Hierarchy → **3D Object → Plane** → rename `floor`
2. Position: `(20, -0.5, 20)`, Scale: `(5, 1, 5)`
3. Drag `FloorMat` onto it

---

## Step 5 — Maze generator

Create `MazeGenerator.cs` in `Scripts/`. Create an empty GameObject named `MazeGenerator` and attach the script.

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
        Vector3 pos = new Vector3(2 * cellSize, 0.5f, 2 * cellSize);
        Instantiate(playerPrefab, pos, Quaternion.identity);
    }

    void PlaceEnemy()
    {
        Vector3 pos = new Vector3((width - 3) * cellSize, 0.5f, (height - 3) * cellSize);
        Instantiate(enemyPrefab, pos, Quaternion.identity);
    }
}
```

After attaching, drag prefabs into the Inspector slots on the MazeGenerator GameObject.

**Solid when:** press Play and a maze generates with blue walls, yellow dots, green player, red enemy.

---

## Step 6 — Player movement

Create `PlayerMovement.cs` in `Scripts/` and attach to the **Player prefab**:

```csharp
using UnityEngine;
using UnityEngine.InputSystem;

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
        var kb = Keyboard.current;
        if (kb == null) return;

        Vector3 direction = Vector3.zero;
        if (kb.wKey.isPressed || kb.upArrowKey.isPressed) direction.z += 1;
        if (kb.sKey.isPressed || kb.downArrowKey.isPressed) direction.z -= 1;
        if (kb.aKey.isPressed || kb.leftArrowKey.isPressed) direction.x -= 1;
        if (kb.dKey.isPressed || kb.rightArrowKey.isPressed) direction.x += 1;

        rb.linearVelocity = new Vector3(direction.normalized.x * moveSpeed, rb.linearVelocity.y, direction.normalized.z * moveSpeed);
    }
}
```

**Solid when:** player moves through corridors with WASD, stops cleanly, can't walk through walls. Recommended `moveSpeed = 5`.

---

## Step 7 — Dot collection

Create `DotCollector.cs` in `Scripts/` and attach to the **Player prefab**:

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
        }
    }
}
```

**Solid when:** walking near a dot destroys it and the count increments.

---

## Step 8 — Enemy chase

Create `EnemyChase.cs` in `Scripts/` and attach to the **Enemy prefab**:

```csharp
using UnityEngine;

public class EnemyChase : MonoBehaviour
{
    public float chaseSpeed = 2.25f;
    public float avoidDistance = 1.5f;

    private Rigidbody rb;

    void Awake()
    {
        rb = GetComponent<Rigidbody>();
    }

    void FixedUpdate()
    {
        GameObject p = GameObject.FindWithTag("Player");
        if (p == null) return;

        Vector3 direction = (p.transform.position - transform.position).normalized;
        direction.y = 0;

        Vector3[] checkDirs = { transform.forward, transform.right, -transform.right };
        foreach (Vector3 dir in checkDirs)
        {
            if (Physics.Raycast(transform.position, dir, avoidDistance))
                direction -= dir * 0.5f;
        }

        direction.y = 0;
        direction.Normalize();

        rb.linearVelocity = new Vector3(direction.x * chaseSpeed, rb.linearVelocity.y, direction.z * chaseSpeed);
    }
}
```

**Solid when:** enemy chases the player, beatable but threatening. Note: enemy may occasionally get briefly stuck at dead ends — full NavMesh pathfinding is a future week.

---

## Step 9 — UI setup

1. Right-click Canvas → **UI → Text - TextMeshPro** → rename `LivesText` → move to top left corner
2. Right-click Canvas → **UI → Text - TextMeshPro** → rename `TimerText` → move to top right corner
3. Right-click Canvas → **UI → Panel** → rename `GameOverPanel`
4. Right-click `GameOverPanel` → **UI → Text - TextMeshPro** → rename `GameOverText` → type "GAME OVER", font size 72, centered
5. Right-click `GameOverPanel` → **UI → Text - TextMeshPro** → rename `FinalScoreText` → center below GameOverText
6. Right-click `GameOverPanel` → **UI → Button - TextMeshPro** → rename `RestartButton` → set text to "Restart"
7. **Disable GameOverPanel** — uncheck the checkbox next to it in the Inspector

Set Canvas **Render Mode** to **Screen Space - Camera** and assign the Main Camera.

---

## Step 10 — Game manager

Create `GameManager.cs` in `Scripts/` and attach to a new empty GameObject named `GameManager`:

```csharp
using UnityEngine;
using UnityEngine.SceneManagement;
using TMPro;

public class GameManager : MonoBehaviour
{
    public static GameManager instance;

    public int lives = 3;
    public float timer = 120f;

    public TextMeshProUGUI livesText;
    public TextMeshProUGUI timerText;
    public TextMeshProUGUI finalScoreText;

    public GameObject gameOverPanel;

    private bool gameOver = false;

    void Awake()
    {
        instance = this;
    }

    void Update()
    {
        if (gameOver) return;

        timer -= Time.deltaTime;
        if (timer <= 0)
        {
            timer = 0;
            TriggerGameOver("Time's Up!");
        }

        timerText.text = "Time: " + Mathf.CeilToInt(timer);
        livesText.text = "Lives: " + lives;
    }

    public void LoseLife()
    {
        if (gameOver) return;
        lives--;
        livesText.text = "Lives: " + lives;
        if (lives <= 0)
            TriggerGameOver("Out of Lives!");
        else
            RespawnPlayer();
    }

    void RespawnPlayer()
    {
        GameObject player = GameObject.FindWithTag("Player");
        if (player != null)
            player.transform.position = new Vector3(4f, 0.5f, 4f);
    }

    void TriggerGameOver(string reason)
    {
        gameOver = true;
        int score = FindFirstObjectByType<DotCollector>()?.dotsCollected ?? 0;
        finalScoreText.text = reason + "\nDots collected: " + score;
        gameOverPanel.SetActive(true);
        Time.timeScale = 0f;
    }

    public void Restart()
    {
        Time.timeScale = 1f;
        SceneManager.LoadScene(SceneManager.GetActiveScene().name);
    }
}
```

Wire up in Inspector:
- Drag `LivesText` → Lives Text slot
- Drag `TimerText` → Timer Text slot
- Drag `FinalScoreText` → Final Score Text slot
- Drag `GameOverPanel` → Game Over Panel slot
- Select `RestartButton` → **On Click ()** → `+` → drag `GameManager` GameObject → select `GameManager.Restart`

---

## Step 11 — Player life and win condition

Create `PlayerLife.cs` in `Scripts/` and attach to the **Player prefab**:

```csharp
using UnityEngine;

public class PlayerLife : MonoBehaviour
{
    private bool invincible = false;

    void OnCollisionEnter(Collision collision)
    {
        if (collision.gameObject.CompareTag("Enemy") && !invincible)
        {
            invincible = true;
            GameManager.instance.LoseLife();
            Invoke(nameof(ResetInvincible), 2f);
        }
    }

    void ResetInvincible()
    {
        invincible = false;
    }
}
```

Create `WinCondition.cs` in `Scripts/` and attach to the **MazeGenerator** GameObject:

```csharp
using UnityEngine;
using UnityEngine.SceneManagement;

public class WinCondition : MonoBehaviour
{
    private int totalDots = 0;
    private DotCollector collector;
    private bool won = false;

    void Start()
    {
        Invoke(nameof(CountDots), 0.5f);
    }

    void CountDots()
    {
        totalDots = FindObjectsByType<SphereCollider>(FindObjectsSortMode.None).Length;
        collector = FindFirstObjectByType<DotCollector>();
    }

    void Update()
    {
        if (won || collector == null || totalDots == 0) return;
        if (collector.dotsCollected >= totalDots)
        {
            won = true;
            Invoke(nameof(LoadNewMaze), 1f);
        }
    }

    void LoadNewMaze()
    {
        SceneManager.LoadScene(SceneManager.GetActiveScene().name);
    }
}
```

**Solid when:** collecting all dots reloads a fresh maze. Losing a life teleports you back to start. Running out of lives or time shows the Game Over screen with a restart button.

---

## Locked values

| Setting | Value |
|---|---|
| moveSpeed | 5 |
| chaseSpeed | 2.25 |
| Maze size | 21x21 |
| Cell size | 2 |
| Starting lives | 3 |
| Timer | 120 seconds |
| Camera orthographic size | 25 |

---

## Common issues

| Problem | Fix |
|---|---|
| Maze doesn't generate | Make sure all four prefabs are assigned in the MazeGenerator Inspector slots |
| Player or enemy sticks to walls | Apply `SlipperyMat` physics material to Player, Enemy, and Wall prefab colliders |
| Dots not disappearing | Check Dot prefab has Is Trigger checked and is tagged `Dot` |
| Enemy gets stuck at dead ends | Expected — full NavMesh pathfinding is a future week |
| Camera doesn't see full maze | Set Camera Position to `(20, 30, 20)`, Orthographic Size to `25` |
| Game Over panel shows at start | Make sure GameOverPanel is disabled (unchecked) in the Inspector before playing |
| Timer not showing | Make sure TimerText is wired into the GameManager Inspector slot |
