# Week 01 — Top-Down Survivor

Build a one-screen top-down shooter in Unity 2D. No art assets — primitives only. By the end you'll have a moving, aiming, shooting player, an enemy that chases you, kill detection, and a score display.

---

## What you need

- Unity 2022+ with the **Universal 2D** template
- TextMeshPro (Unity will prompt you to import it)

---

## Project setup

1. Create a new Unity project using the **Universal 2D** template
2. In the Hierarchy, confirm you have a **Main Camera** and a **Global Light 2D** — the 2D template includes these automatically

---

## Step 1 — Player setup (Editor)

1. Right-click Hierarchy → **2D Object → Sprites → Square** → rename it `Player`
2. Add component: **Rigidbody2D** → set **Gravity Scale to 0**
3. Set tag to **Player** (Edit → Project Settings → Tags and Layers to add it first)

---

## Step 2 — Enemy prefab (Editor)

1. Right-click Hierarchy → **2D Object → Sprites → Circle** → rename it `Enemy`
2. Add **Rigidbody2D** (Gravity Scale 0) and **CircleCollider2D**
3. Tag it **Enemy**
4. Drag it from the Hierarchy into the Project panel to save as a prefab
5. Delete it from the scene

---

## Step 3 — Projectile prefab (Editor)

1. Right-click Hierarchy → **2D Object → Sprites → Capsule** → rename it `Projectile`
2. Add **Rigidbody2D** (Gravity Scale 0) and **CapsuleCollider2D** → check **Is Trigger**
3. Tag it **Projectile**
4. Drag to Project panel to save as prefab → delete from scene

---

## Step 4 — Score UI (Editor)

1. Right-click Hierarchy → **UI (Canvas) → Text - TextMeshPro** → rename it `ScoreText`
2. Click **Import TMP Essentials** if prompted
3. Position it in a corner of the Canvas

---

## Step 5 — Player movement

Create a C# script named `PlayerMovement`, paste this in, and attach it to the **Player** GameObject:

```csharp
using UnityEngine;
using UnityEngine.InputSystem;

public class PlayerMovement : MonoBehaviour
{
    public float moveSpeed = 5f;

    private Rigidbody2D rb;
    private Vector2 movement;

    void Awake()
    {
        rb = GetComponent<Rigidbody2D>();
    }

    void Update()
    {
        var kb = Keyboard.current;
        if (kb == null) return;

        movement = Vector2.zero;
        if (kb.wKey.isPressed || kb.upArrowKey.isPressed) movement.y += 1;
        if (kb.sKey.isPressed || kb.downArrowKey.isPressed) movement.y -= 1;
        if (kb.aKey.isPressed || kb.leftArrowKey.isPressed) movement.x -= 1;
        if (kb.dKey.isPressed || kb.rightArrowKey.isPressed) movement.x += 1;
    }

    void FixedUpdate()
    {
        rb.linearVelocity = movement.normalized * moveSpeed;
    }
}
```

**Solid when:** player moves all four directions, stops cleanly on key release, speed feels controllable. Recommended `moveSpeed = 5`.

---

## Step 6 — Player aim

Create `PlayerAim.cs`, attach to **Player**:

```csharp
using UnityEngine;
using UnityEngine.InputSystem;

public class PlayerAim : MonoBehaviour
{
    private Rigidbody2D rb;

    void Awake()
    {
        rb = GetComponent<Rigidbody2D>();
    }

    void Update()
    {
        if (Mouse.current == null) return;
        Vector3 mousePos = Camera.main.ScreenToWorldPoint(Mouse.current.position.ReadValue());
        mousePos.z = transform.position.z;
        Vector3 direction = mousePos - transform.position;
        float angle = Mathf.Atan2(direction.y, direction.x) * Mathf.Rad2Deg;
        rb.MoveRotation(angle);
    }
}
```

**Solid when:** player rotates to face the mouse cursor smoothly with no jitter.

---

## Step 7 — Shooting

Create `PlayerShoot.cs`, attach to **Player**. Then create `Projectile.cs` and attach it to the **Projectile prefab**.

**PlayerShoot.cs:**
```csharp
using UnityEngine;
using UnityEngine.InputSystem;

public class PlayerShoot : MonoBehaviour
{
    public GameObject projectilePrefab;
    public float projectileSpeed = 10f;
    public float fireRate = 2.25f;

    private float nextFireTime = 0f;

    void Update()
    {
        if (Mouse.current == null) return;
        if (Mouse.current.leftButton.isPressed && Time.time >= nextFireTime)
        {
            nextFireTime = Time.time + 1f / fireRate;
            GameObject p = Instantiate(projectilePrefab, transform.position, transform.rotation);
            p.GetComponent<Rigidbody2D>().linearVelocity = transform.right * projectileSpeed;
        }
    }
}
```

**Projectile.cs:**
```csharp
using UnityEngine;

public class Projectile : MonoBehaviour
{
    public float lifetime = 2f;

    void Start()
    {
        Destroy(gameObject, lifetime);
    }
}
```

After attaching the scripts:
- Select the **Player** → find the `PlayerShoot` component → drag the **Projectile prefab** into the `Projectile Prefab` slot

**Solid when:** left click fires in the aim direction, fire rate feels neither machine-gun nor sluggish, projectiles despawn. Recommended `fireRate = 2.25`.

---

## Step 8 — Enemy chase

Create `EnemyChase.cs`, attach to the **Enemy prefab**:

```csharp
using UnityEngine;

public class EnemyChase : MonoBehaviour
{
    public float chaseSpeed = 2.25f;

    private Transform player;

    void Start()
    {
        player = GameObject.FindWithTag("Player").transform;
    }

    void FixedUpdate()
    {
        if (player == null) return;
        Vector2 direction = (player.position - transform.position).normalized;
        GetComponent<Rigidbody2D>().linearVelocity = direction * chaseSpeed;
    }
}
```

Drag one Enemy prefab into the scene to test. **Solid when:** enemy chases smoothly, beatable but threatening. Recommended `chaseSpeed = 2.25`.

---

## Step 9 — Kill and respawn

Create `EnemyHealth.cs`, attach to the **Enemy prefab**:

```csharp
using UnityEngine;

public class EnemyHealth : MonoBehaviour
{
    void OnTriggerEnter2D(Collider2D other)
    {
        if (other.CompareTag("Projectile"))
        {
            Destroy(other.gameObject);
            if (ScoreManager.instance != null)
                ScoreManager.instance.AddKill();
            Destroy(gameObject);
        }
    }
}
```

Create `EnemySpawner.cs`, attach to a new empty GameObject named `Spawner`:

```csharp
using UnityEngine;

public class EnemySpawner : MonoBehaviour
{
    public GameObject enemyPrefab;
    private GameObject currentEnemy;

    void Start()
    {
        SpawnEnemy();
    }

    void Update()
    {
        if (currentEnemy == null)
            SpawnEnemy();
    }

    void SpawnEnemy()
    {
        Vector2 spawnPos = GetScreenEdgePosition();
        currentEnemy = Instantiate(enemyPrefab, spawnPos, Quaternion.identity);
    }

    Vector2 GetScreenEdgePosition()
    {
        Camera cam = Camera.main;
        float height = cam.orthographicSize;
        float width = height * cam.aspect;

        int edge = Random.Range(0, 4);
        return edge switch
        {
            0 => new Vector2(Random.Range(-width, width), height + 1),
            1 => new Vector2(Random.Range(-width, width), -height - 1),
            2 => new Vector2(-width - 1, Random.Range(-height, height)),
            _ => new Vector2(width + 1, Random.Range(-height, height)),
        };
    }
}
```

Drag the **Enemy prefab** into the `Enemy Prefab` slot on the Spawner. Delete the test enemy from the scene.

**Solid when:** projectile destroys the enemy on contact, exactly one enemy alive at a time, replacement spawns within ~1 second.

---

## Step 10 — Score

Create `ScoreManager.cs`, attach to a new empty GameObject named `ScoreManager`:

```csharp
using UnityEngine;
using TMPro;

public class ScoreManager : MonoBehaviour
{
    public TextMeshProUGUI scoreText;
    private int score = 0;

    public static ScoreManager instance;

    void Awake()
    {
        instance = this;
    }

    public void AddKill()
    {
        score++;
        scoreText.text = "Score: " + score;
    }
}
```

Drag **ScoreText** from the Hierarchy into the `Score Text` slot on the ScoreManager component.

**Solid when:** score increments exactly once per kill and displays correctly on screen.

---

## Locked values

| Setting | Value |
|---|---|
| moveSpeed | 5 |
| fireRate | 2.25 |
| chaseSpeed | 2.25 |

---

## Common issues

| Problem | Fix |
|---|---|
| Input error about UnityEngine.Input | Project uses new Input System — use `UnityEngine.InputSystem` as shown in the scripts above |
| Player not rotating | Make sure `PlayerAim` is attached and enabled on the Player |
| Projectile falls with gravity | Set Gravity Scale to 0 on the Projectile's Rigidbody2D |
| Shooting does nothing to enemy | Check: EnemyHealth is on the Enemy prefab (not just the scene instance), Projectile has Is Trigger checked, Enemy collider does NOT have Is Trigger |
| Score not updating | Make sure ScoreText is wired into the ScoreManager component's Score Text slot |
