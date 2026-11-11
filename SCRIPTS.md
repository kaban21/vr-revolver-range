# Все скрипты проекта одним файлом

Порядок: Weapon, Interaction, Range, Player. Каждый блок = один файл в Assets/Scripts/<папка>/.

## Weapon/Bullet.cs

```csharp
using UnityEngine;

// Псевдобаллистика: рейкаст по сегменту пути каждый FixedUpdate (90 Гц, шаг ≈3.3 м), гравитация,
// чтобы пуля не пролетала сквозь мишень. Падение ≈14 см на 50 м при 300 м/с.
// Префаб: пустой объект + TrailRenderer (время 0.08 с). Меш не нужен.
public class Bullet : MonoBehaviour
{
    public float speed = 300f;
    public float life = 3f;
    public float impulse = 3f;                 // Н·с по физическим телам (≈10 г × 300 м/с)
    public LayerMask mask;                     // Target | Env
    public TrailRenderer trail;

    Vector3 _vel;
    float _t;
    bool _live;

    public void Launch(Vector3 pos, Vector3 dir)
    {
        transform.position = pos;
        _vel = dir.normalized * speed;
        _t = 0f;
        _live = true;
        gameObject.SetActive(true);
        if (trail) trail.Clear();
    }

    void FixedUpdate()
    {
        if (!_live) return;
        float dt = Time.fixedDeltaTime;
        _vel += Physics.gravity * dt;
        Vector3 step = _vel * dt;

        if (Physics.Raycast(transform.position, step.normalized, out RaycastHit hit, step.magnitude, mask, QueryTriggerInteraction.Ignore))
        {
            IShootable s = hit.collider.GetComponentInParent<IShootable>();
            if (s != null) s.OnHit(hit, _vel);
            if (hit.rigidbody) hit.rigidbody.AddForceAtPosition(_vel.normalized * impulse, hit.point, ForceMode.Impulse);
            Despawn();
            return;
        }

        transform.position += step;
        _t += dt;
        if (_t > life) Despawn();
    }

    void Despawn() { _live = false; gameObject.SetActive(false); }
}
```

## Weapon/BulletPool.cs

```csharp
using UnityEngine;

// Пул пуль (32 штуки). Одна копия на сцену.
public class BulletPool : MonoBehaviour
{
    public Bullet prefab;
    public int size = 32;

    static BulletPool _instance;
    Bullet[] _pool;
    int _next;

    void Awake()
    {
        _instance = this;
        _pool = new Bullet[size];
        for (int i = 0; i < size; i++)
        {
            _pool[i] = Instantiate(prefab, transform);
            _pool[i].gameObject.SetActive(false);
        }
    }

    void OnDestroy() { if (_instance == this) _instance = null; }

    public static Bullet Get()
    {
        if (_instance == null) return null;
        var b = _instance._pool[_instance._next];
        _instance._next = (_instance._next + 1) % _instance._pool.Length;
        return b;                                  // при переполнении переиспользуем самую старую
    }
}
```

## Weapon/Cartridge.cs

```csharp
using UnityEngine;
using UnityEngine.XR.Interaction.Toolkit;
using UnityEngine.XR.Interaction.Toolkit.Interactables;

public enum CartState { Live, Spent }

// Патрон / гильза: отдельный физический объект. Пивот в центре донца, +Z смотрит на пулю.
// Rigidbody: масса 0.015, ContinuousDynamic, maxDepenetrationVelocity 1.5.
// XRGrabInteractable: Velocity Tracking, Velocity Scale 1.0, Damping 0.7, Throw On Detach = on.
// В colliders интерактабла добавь невидимую сферу-триггер R=0.02 (зона захвата).
[RequireComponent(typeof(Rigidbody), typeof(XRGrabInteractable))]
public class Cartridge : MonoBehaviour
{
    public CartState state = CartState.Live;
    public GameObject bulletMesh;           // латунь остаётся, пуля исчезает после выстрела
    public Renderer primer;
    public Material dentedPrimer;           // капсюль с вмятиной
    public Collider[] bodyColliders;        // физические коллайдеры (не зона захвата)

    [Header("Возврат на стол")]
    public Vector3 home;                    // задаёт AmmoTable
    public bool hasHome;
    public float floorY = 0.15f;            // ниже этого считаем, что упал на пол
    public float returnAfter = 10f;         // через сколько секунд вернуть на стол
    public float spentLifetime = 20f;       // через сколько секунд исчезает выброшенная гильза

    [HideInInspector] public Chamber seated;
    public Rigidbody Rb { get; private set; }
    public XRGrabInteractable Grab { get; private set; }

    HandRef _hand;
    float _idleTimer, _lastSound;

    void Awake()
    {
        Rb = GetComponent<Rigidbody>();
        Grab = GetComponent<XRGrabInteractable>();
        Rb.mass = 0.015f;
        Rb.maxDepenetrationVelocity = 1.5f;
        Rb.sleepThreshold = 0.05f;
        Rb.collisionDetectionMode = CollisionDetectionMode.ContinuousDynamic;
        if (bodyColliders == null || bodyColliders.Length == 0) bodyColliders = GetComponentsInChildren<Collider>();
    }

    void OnEnable()
    {
        Grab.selectEntered.AddListener(OnGrab);
        Grab.selectExited.AddListener(OnDrop);
    }

    void OnDisable()
    {
        Grab.selectEntered.RemoveListener(OnGrab);
        Grab.selectExited.RemoveListener(OnDrop);
    }

    // Пока патрон в руке, он не должен толкаться с коллайдерами этой же руки (иначе дрожит).
    void OnGrab(SelectEnterEventArgs a) { _hand = HandRef.Of(a.interactorObject); SetIgnore(true); }
    void OnDrop(SelectExitEventArgs a) { SetIgnore(false); _hand = null; }

    void SetIgnore(bool ignore)
    {
        if (_hand == null) return;
        foreach (var c in bodyColliders)
            foreach (var h in _hand.colliders)
                if (c && h) Physics.IgnoreCollision(c, h, ignore);
    }

    public void Fire()
    {
        state = CartState.Spent;
        if (bulletMesh) bulletMesh.SetActive(false);
        if (primer && dentedPrimer) primer.sharedMaterial = dentedPrimer;
    }

    // Посадка в камору: Chamber уже отпустил руку
    public void SeatIn(Chamber ch)
    {
        Grab.enabled = false;
        Rb.isKinematic = true;
        Rb.detectCollisions = false;
        seated = ch;
        _idleTimer = 0f;
    }

    public void Eject(Vector3 dir, float speed)
    {
        seated = null;
        transform.SetParent(null, true);
        Rb.isKinematic = false;
        Rb.detectCollisions = true;
        Grab.enabled = true;
        Rb.linearVelocity = dir * speed;
        Rb.angularVelocity = Random.insideUnitSphere * 4f;
        _idleTimer = 0f;
    }

    void Update()
    {
        if (seated || Grab.isSelected) { _idleTimer = 0f; return; }

        // Гильза без дела исчезает, живой патрон на полу возвращается на стол
        bool onFloor = transform.position.y < floorY;
        if (state == CartState.Spent) { _idleTimer += Time.deltaTime; if (_idleTimer > spentLifetime) Destroy(gameObject); return; }
        if (!onFloor || !hasHome) { _idleTimer = 0f; return; }
        _idleTimer += Time.deltaTime;
        if (_idleTimer > returnAfter)
        {
            _idleTimer = 0f;
            Rb.linearVelocity = Vector3.zero; Rb.angularVelocity = Vector3.zero;
            transform.SetPositionAndRotation(home, Quaternion.Euler(0f, Random.Range(0f, 360f), 0f));   // лёжа на боку: ось Z горизонтальна
        }
    }

    void OnCollisionEnter(Collision c)
    {
        float v = c.relativeVelocity.magnitude;
        if (v < 0.4f || Time.time - _lastSound < 0.05f) return;
        _lastSound = Time.time;
        if (RevolverAudio.Instance) RevolverAudio.Instance.CasingDrop(transform.position, v);
    }
}
```

## Weapon/Chamber.cs

```csharp
using System.Collections;
using UnityEngine;
using UnityEngine.XR.Interaction.Toolkit.Interactables;

// Одна камора барабана. Ребёнок rotor. На этом же объекте: SphereCollider (isTrigger) R≈0.012,
// сдвинутый на 0.01 назад от устья. mouth: начало в центре заднего среза каморы,
// mouth.forward смотрит ВГЛУБЬ каморы (к стволу). Патрон вставляется сзади.
public class Chamber : MonoBehaviour
{
    public Transform mouth;
    public Cylinder cylinder;
    public Cartridge occupant;

    [Header("Допуски вставки")]
    public float maxAngle = 25f;        // угол между осью патрона и осью каморы
    public float maxLateral = 0.007f;   // боковое смещение, м
    public float maxApproach = 0.015f;  // насколько далеко снаружи ещё считаем «у устья»
    public float seatDepth = -0.003f;   // донце дошло до среза: садим
    public float slideTime = 0.06f;

    void OnTriggerStay(Collider other)
    {
        if (!cylinder.IsOpen || occupant) return;
        var c = other.GetComponentInParent<Cartridge>();
        if (!c || c.seated || !c.Grab.isSelected) return;         // вставляем только из руки

        Vector3 d = c.transform.position - mouth.position;
        float depth = Vector3.Dot(d, mouth.forward);              // <0: снаружи, 0: донце у среза
        float lateral = (d - mouth.forward * depth).magnitude;

        if (Vector3.Angle(c.transform.forward, mouth.forward) > maxAngle) return;
        if (lateral > maxLateral || depth < -maxApproach || depth > 0.04f) return;
        if (depth >= seatDepth) Seat(c);
    }

    void Seat(Cartridge c)
    {
        var hand = HandRef.Of(c.Grab.firstInteractorSelecting);
        c.Grab.interactionManager.CancelInteractableSelection((IXRSelectInteractable)c.Grab);
        c.SeatIn(this);
        occupant = c;
        c.transform.SetParent(mouth, true);
        StartCoroutine(Slide(c.transform));
        cylinder.sfx.Seat(mouth.position);
        Haptics.Play(hand, 0.3f, 0.02f);
    }

    IEnumerator Slide(Transform t)      // 60 мс до идеальной позы внутри каморы
    {
        Vector3 p0 = t.localPosition;
        Quaternion r0 = t.localRotation;
        for (float k = 0f; k < 1f; k += Time.deltaTime / slideTime)
        {
            if (!t) yield break;
            t.localPosition = Vector3.Lerp(p0, Vector3.zero, k);
            t.localRotation = Quaternion.Slerp(r0, Quaternion.identity, k);
            yield return null;
        }
        if (t) { t.localPosition = Vector3.zero; t.localRotation = Quaternion.identity; }
    }

    public void Eject(float speed)
    {
        var c = occupant;
        occupant = null;
        if (c) c.Eject(-mouth.forward, speed);                    // выброс назад, через задний срез
    }
}
```

## Weapon/Cylinder.cs

```csharp
using System;
using System.Collections;
using UnityEngine;

// Барабан на откидном журавле (swing-out, открывается влево).
// Иерархия: Revolver / Frame / SwingPivot / Rotor / Chamber x6
//  - SwingPivot: ось шарнира ‖ стволу (локальная Z), прямо ПОД осью барабана, вращается на SwingAngle (0..95°)
//  - Rotor: вращается вокруг локальной Z (ось ствола); шаг 60°
// Журавль двигается скриптом, не физическим HingeJoint: так нет дрожи при движении руки в VR.
public class Cylinder : MonoBehaviour
{
    [Header("Ссылки")]
    public Chamber[] chambers;          // 6 штук по кругу
    public Transform rotor, swingPivot;
    public Transform strikePoint;       // на оси ствола у заднего среза барабана (ребёнок рамки, не ротора)
    public Transform gun;               // корень револьвера
    public Hammer hammer;
    public RevolverAudio sfx;

    [Header("Параметры")]
    public float openLimit = 95f;
    public float openAngle = 4f;        // больше этого барабан считается открытым
    public float lockAngle = 3f;        // меньше этого при закрытии срабатывает фиксатор
    public float turnTime = 0.04f;      // поворот на 60° при взводе
    public float ejectSpeed = 0.35f;    // м/с выброс гильз назад
    public float shakeEject = 8f;       // м/с², тряска, при которой «залипшие» гильзы вылетают

    public float SwingAngle { get; private set; }
    public bool IsOpen => SwingAngle > openAngle;
    public bool IsLatched => _latched;
    public bool IsTurning { get; private set; }

    public event Action OnOpened, OnClosed;

    bool _latched = true, _wasOpen, _driven, _ejectorArmed = true;
    float _driveTarget, _swingVel, _spinAcc, _shake, _nextDrop;
    Quaternion _baseRot;
    Vector3 _prevPos, _prevVel;

    void Awake()
    {
        _baseRot = rotor.localRotation;
        _prevPos = gun.position;
    }

    // Камора, которая сейчас стоит напротив бойка и ствола
    public Chamber Current
    {
        get
        {
            Chamber best = null; float bd = float.MaxValue;
            foreach (var c in chambers)
            {
                float d = (c.mouth.position - strikePoint.position).sqrMagnitude;
                if (d < bd) { bd = d; best = c; }
            }
            return best;
        }
    }

    // Защёлка (PushSlider, значение 0..1). Порог 80%.
    public void SetLatch(float v)
    {
        if (_latched && v >= 0.8f && hammer.IsDown)
        {
            _latched = false;
            _swingVel = 50f;            // небольшой «выскок» журавля: ~5°, барабан считается открытым
            sfx.Latch();
        }
    }

    // Рука тянет журавль: target = желаемый угол, считает CylinderHandle
    public void DriveSwing(float target)
    {
        if (_latched) return;
        _driven = true;
        _driveTarget = Mathf.Clamp(target, 0f, openLimit);
    }

    // Курок зовёт на ~70% хода взвода
    public void Advance() { if (_latched && !IsOpen) StartCoroutine(Turn(60f)); }

    IEnumerator Turn(float deg)
    {
        IsTurning = true;
        Quaternion a = _baseRot, b = a * Quaternion.Euler(0f, 0f, deg);
        for (float k = 0f; k < 1f; k += Time.deltaTime / turnTime)
        {
            rotor.localRotation = Quaternion.Slerp(a, b, k);
            yield return null;
        }
        _baseRot = b;
        rotor.localRotation = b;
        IsTurning = false;
    }

    // Открытый барабан крутят рукой: щелчок и фиксация каждые 60°.
    // Между щелчками ротор следует за рукой на 50%, чтобы чувствовался «упор».
    public void SpinByHand(float deltaDeg, HandRef hand)
    {
        if (!IsOpen) return;
        _spinAcc += deltaDeg;
        while (Mathf.Abs(_spinAcc) >= 60f)
        {
            float s = Mathf.Sign(_spinAcc);
            _baseRot *= Quaternion.Euler(0f, 0f, 60f * s);
            _spinAcc -= 60f * s;
            sfx.CylinderTick();
            Haptics.Play(hand, 0.25f, 0.015f);
        }
        rotor.localRotation = _baseRot * Quaternion.Euler(0f, 0f, _spinAcc * 0.5f);
    }

    public void EndSpin()
    {
        if (!IsOpen) return;
        _spinAcc = 0f;
        rotor.localRotation = _baseRot;
    }

    // Шток экстрактора (PushSlider): толкнули до конца: стреляные гильзы вылетают
    public void PushEjector(float v)
    {
        if (_ejectorArmed && v >= 0.9f && IsOpen) { _ejectorArmed = false; EjectSpent(); }
        else if (!_ejectorArmed && v <= 0.3f) _ejectorArmed = true;
    }

    public void EjectSpent()
    {
        foreach (var c in chambers)
            if (c.occupant && c.occupant.state == CartState.Spent) c.Eject(ejectSpeed);
        sfx.EjectRod();
    }

    void Update()
    {
        float dt = Time.deltaTime;
        float prev = SwingAngle;

        if (_driven)
        {
            SwingAngle = Mathf.Lerp(SwingAngle, _driveTarget, 1f - Mathf.Exp(-30f * dt));
            if (dt > 0f) _swingVel = (SwingAngle - prev) / dt;
            _driven = false;
        }
        else if (!_latched)
        {
            SwingAngle += _swingVel * dt;
            _swingVel *= Mathf.Exp(-8f * dt);       // трение шарнира
        }

        float max = _latched ? 0f : openLimit;
        if (SwingAngle > max) { SwingAngle = max; if (_swingVel > 0f) _swingVel *= -0.15f; }   // упор
        if (SwingAngle < 0f) { SwingAngle = 0f; if (_swingVel < 0f) _swingVel = 0f; }

        // Закрыли: «клац» и блокировка
        if (!_latched && _wasOpen && SwingAngle <= lockAngle)
        {
            _latched = true; SwingAngle = 0f; _swingVel = 0f; _wasOpen = false;
            sfx.CylinderClose();
            OnClosed?.Invoke();
        }

        swingPivot.localRotation = Quaternion.Euler(0f, 0f, SwingAngle);

        if (IsOpen && !_wasOpen) { _wasOpen = true; OnOpened?.Invoke(); }

        hammer.Locked = !_latched || IsOpen;     // курок нельзя взводить при расфиксированном/открытом барабане
    }

    void FixedUpdate()
    {
        float dt = Time.fixedDeltaTime;
        Vector3 v = (gun.position - _prevPos) / dt;
        _shake = Mathf.Lerp(_shake, (v - _prevVel).magnitude / dt, 0.3f);
        _prevPos = gun.position; _prevVel = v;

        if (!IsOpen || Time.time < _nextDrop) return;

        bool muzzleUp = Vector3.Dot(gun.forward, Vector3.up) > 0.7f;   // ствол вверх ≥ ~45°: гравитация выводит патроны назад
        foreach (var c in chambers)
        {
            if (!c.occupant) continue;
            bool live = c.occupant.state == CartState.Live;
            if ((muzzleUp && live) || _shake > shakeEject)
            {
                c.Eject(0.15f);
                _nextDrop = Time.time + 0.08f;      // вылетают по одному, а не горстью
                break;
            }
        }
    }
}
```

## Weapon/Hammer.cs

```csharp
using System;
using UnityEngine;

// Курок. Ось вращения: локальная X пивота, в покое pivot.localRotation = identity,
// шпора смотрит вверх (+Y), при взводе уходит назад (-Z).
// Углы: 0° покой, 49° сухарь цепляет, 52° боевой взвод.
// Не доведённый до 49° курок пружиной (600°/с) возвращается без выстрела.
public class Hammer : MonoBehaviour
{
    public Transform pivot;
    public const float Full = 52f, SearCatch = 49f, IndexAt = 36f, Notch = 13f;
    public float dragSpeed = 900f;      // °/с, как быстро курок догоняет руку
    public float returnSpeed = 600f;    // °/с, пружина возврата
    public bool Locked;                 // true, пока барабан открыт

    public float Angle { get; private set; }
    public bool IsCocked { get; private set; }
    public bool Decocking { get; private set; }
    public bool IsHeld => _owner != null;
    public bool IsDown => !IsCocked && Angle < 1f;

    public event Action<int> OnNotch;                       // щелчок трещотки (1..4)
    public event Action OnIndex, OnCocked, OnSlipped, OnStrike;

    Quaternion _rest;
    object _owner;
    int _lastNotch;
    bool _indexed;

    void Awake() { _rest = pivot.localRotation; }

    // Угол курка, на который указывает точка мира (кончик большого пальца и т.п.).
    public float AngleFromPoint(Vector3 world)
    {
        Vector3 p = pivot.parent.InverseTransformPoint(world) - pivot.localPosition;
        return Mathf.Atan2(-p.z, p.y) * Mathf.Rad2Deg;
    }

    // Владелец (рука или палец) берёт курок. Одновременно держать может только один.
    public bool TryHold(object owner)
    {
        if (Locked) return false;
        if (_owner == null || _owner == owner) { _owner = owner; return true; }
        return false;
    }

    public void Release(object owner)
    {
        if (_owner != owner) return;
        _owner = null;
        Decocking = false;
    }

    // Нажали спуск, пока курок в руке: курок идёт вниз за рукой без выстрела.
    public void BeginDecock()
    {
        if (_owner != null && IsCocked) { IsCocked = false; Decocking = true; }
    }

    // Вызывается каждый кадр, пока рука ведёт курок
    public void Drag(object owner, float targetAngle)
    {
        if (_owner != owner || Locked) return;
        if (IsCocked && !Decocking) return;                 // уже на боевом взводе

        Angle = Mathf.MoveTowards(Angle, Mathf.Clamp(targetAngle, 0f, Full), dragSpeed * Time.deltaTime);

        bool justCocked = false;
        if (!Decocking && Angle >= SearCatch)
        {
            IsCocked = true; Angle = Full; justCocked = true;
        }
        Apply();

        int n = Mathf.FloorToInt(Angle / Notch);
        if (n != _lastNotch) { _lastNotch = n; if (n > 0) OnNotch?.Invoke(n); }

        if (Decocking)
        {
            if (Angle <= 0.5f) { Angle = 0f; Decocking = false; ResetIndex(); Apply(); OnSlipped?.Invoke(); }
            return;
        }

        if (Angle < 5f) _indexed = false;
        if (!_indexed && Angle >= IndexAt) { _indexed = true; OnIndex?.Invoke(); }   // барабан +60°
        if (justCocked) OnCocked?.Invoke();
    }

    // Сухарь сорван: курок падает на боёк (визуально сразу, удар отдаёт Revolver через 12 мс)
    public void Fall()
    {
        if (!IsCocked) return;
        IsCocked = false; Decocking = false; Angle = 0f;
        ResetIndex(); Apply();
        OnStrike?.Invoke();
    }

    void Update()
    {
        if (IsHeld || IsCocked || Angle <= 0f) return;
        Angle = Mathf.MoveTowards(Angle, 0f, returnSpeed * Time.deltaTime);   // не дожали: падает без выстрела
        Apply();
        _lastNotch = Mathf.FloorToInt(Angle / Notch);
        if (Angle <= 0f) { ResetIndex(); OnSlipped?.Invoke(); }
    }

    void ResetIndex() { _indexed = false; _lastNotch = 0; }
    void Apply() { pivot.localRotation = _rest * Quaternion.Euler(-Angle, 0f, 0f); }
}
```

## Weapon/Haptics.cs

```csharp
using UnityEngine;

// Статический помощник: хаптик на руку одной строкой.
// Амплитуда дополнительно умножается на пользовательскую силу вибрации (ComfortSettings).
public static class Haptics
{
    public static void Play(HandRef hand, float amplitude, float duration)
    {
        if (hand != null) hand.Haptic(amplitude, duration);
    }
}
```

## Weapon/Holster.cs

```csharp
using UnityEngine;
using UnityEngine.XR.Interaction.Toolkit.Interactables;
using UnityEngine.XR.Interaction.Toolkit.Interactors;

// Кобура привязана к ТЕЛУ, а не к голове: рыскание тела догоняет голову с мёртвой зоной,
// поэтому оружие не «улетает» вместе с поворотом головы.
// Сокет (XRSocketInteractor на этом же объекте): Show Interactable Hover Meshes = off (никаких подсказок),
// Interaction Layers = Weapon, радиус 0.05. Оружие доставать и возвращать обычным хватом.
public class Holster : MonoBehaviour
{
    public Transform head, rig;                    // камера и XR Origin (пол)
    public XRSocketInteractor socket;
    public Revolver revolver;
    public float beltRatio = 0.58f;                // высота пояса / высота глаз
    public Vector3 offset = new Vector3(0.20f, 0f, -0.05f);   // вправо и чуть назад от центра тела
    public float pitch = 15f;                      // наклон рукояти вперёд-вверх
    public float yawDeadzone = 25f, yawSpeed = 120f;

    float _yaw, _nextRub;
    bool _follow;
    Vector3 _lastGunPos;

    void Start()
    {
        _yaw = head.eulerAngles.y;
        if (revolver) revolver.Grab.selectEntered.AddListener(OnRevolverSelected);
    }

    void OnDestroy()
    {
        if (revolver) revolver.Grab.selectEntered.RemoveListener(OnRevolverSelected);
    }

    // Рука взяла револьвер: сокет его отпускает (оружие «выходит» из кобуры)
    void OnRevolverSelected(UnityEngine.XR.Interaction.Toolkit.SelectEnterEventArgs a)
    {
        if (a.interactorObject is XRSocketInteractor) return;
        if (socket.hasSelection && socket.IsSelecting((IXRSelectInteractable)revolver.Grab))
            socket.interactionManager.SelectExit(socket, (IXRSelectInteractable)revolver.Grab);
    }

    // Вернуть оружие в кобуру (упавшее на пол)
    public void Stow(Revolver gun)
    {
        if (socket.hasSelection) return;
        socket.StartManualInteraction((IXRSelectInteractable)gun.Grab);
    }

    void LateUpdate()
    {
        float headYaw = head.eulerAngles.y;
        float diff = Mathf.Abs(Mathf.DeltaAngle(_yaw, headYaw));
        if (diff > yawDeadzone) _follow = true;
        else if (diff < 4f) _follow = false;                       // гистерезис
        if (_follow) _yaw = Mathf.MoveTowardsAngle(_yaw, headYaw, yawSpeed * Time.deltaTime);

        float eye = head.position.y - rig.position.y;
        float beltY = rig.position.y + eye * beltRatio + ComfortSettings.BeltOffset;

        Quaternion body = Quaternion.Euler(0f, _yaw, 0f);
        Vector3 off = offset;
        if (!ComfortSettings.DominantRight) off.x = -off.x;        // левша: кобура слева
        Vector3 center = new Vector3(head.position.x, beltY, head.position.z);
        transform.SetPositionAndRotation(center + body * off, body * Quaternion.Euler(pitch, 0f, 0f));

        LeatherRub();
    }

    // Тёрка кожи: рука с револьвером быстро движется рядом с кобурой
    void LeatherRub()
    {
        if (!revolver || !revolver.sfx) return;
        Vector3 p = revolver.transform.position;
        float speed = Time.deltaTime > 0f ? (p - _lastGunPos).magnitude / Time.deltaTime : 0f;
        _lastGunPos = p;
        if (revolver.IsHeld && speed > 0.25f && Time.time > _nextRub && (p - transform.position).sqrMagnitude < 0.18f * 0.18f)
        {
            _nextRub = Time.time + 0.25f;
            revolver.sfx.LeatherRub(transform.position, speed);
        }
    }
}
```

## Weapon/Revolver.cs

```csharp
using System;
using System.Collections;
using UnityEngine;
using UnityEngine.XR.Interaction.Toolkit;
using UnityEngine.XR.Interaction.Toolkit.Interactables;
using UnityEngine.XR.Interaction.Toolkit.Interactors;

// Оркестратор: курок, спуск, барабан, пуля, отдача, хаптика. Остальные детали о нём не знают.
// XRGrabInteractable: Select Mode = Multiple, Throw On Detach = off, Attach Transform = GripPoint,
// Track Position/Rotation выключаются здесь: позу оружия ведёт TwoHandAim (ноль задержки).
// Масса Rigidbody 1.1 кг.
[RequireComponent(typeof(Rigidbody), typeof(XRGrabInteractable))]
public class Revolver : MonoBehaviour
{
    [Header("Детали")]
    public Hammer hammer;
    public RevolverTrigger trigger;
    public Cylinder cylinder;
    public RevolverAudio sfx;
    public Holster holster;                 // необязательно: для возврата упавшего оружия

    [Header("Дуло и эффекты")]
    public Transform muzzle;
    public Transform recoilPivot;           // родитель всей визуальной части (не GripPoint)
    public Light flash;                     // point light, радиус 3 м
    public ParticleSystem muzzleSmoke, gapSmoke;

    [Header("Отдача")]
    public float kickAngle = 8f;            // подброс, °
    public float kickBack = 0.02f;          // откат, м
    public float kickUpTime = 0.035f;
    public float kickReturnTime = 0.22f;
    public float twoHandRecoil = 0.55f;     // множитель отдачи при хвате двумя руками

    public HandRef MainHand { get; private set; }
    public HandRef SupportHand { get; private set; }
    public XRSocketInteractor Socket { get; private set; }
    public XRGrabInteractable Grab { get; private set; }
    public Rigidbody Rb { get; private set; }
    public bool IsHeld => MainHand != null;

    public event Action OnShot;

    float _floorTimer;

    void Awake()
    {
        Grab = GetComponent<XRGrabInteractable>();
        Rb = GetComponent<Rigidbody>();
        Rb.mass = 1.1f;
        Grab.trackPosition = false;
        Grab.trackRotation = false;
        Grab.throwOnDetach = false;
        Grab.selectMode = InteractableSelectMode.Multiple;
        if (flash) flash.enabled = false;
    }

    void OnEnable()
    {
        Grab.selectEntered.AddListener(OnSelected);
        Grab.selectExited.AddListener(OnDeselected);
        hammer.OnNotch += OnNotch;
        hammer.OnIndex += cylinder.Advance;
        hammer.OnCocked += OnCocked;
        hammer.OnSlipped += OnHammerDown;
        trigger.OnBreak += OnSearBreak;
        trigger.OnDeadPull += OnDeadPull;
    }

    void OnDisable()
    {
        Grab.selectEntered.RemoveListener(OnSelected);
        Grab.selectExited.RemoveListener(OnDeselected);
        hammer.OnNotch -= OnNotch;
        hammer.OnIndex -= cylinder.Advance;
        hammer.OnCocked -= OnCocked;
        hammer.OnSlipped -= OnHammerDown;
        trigger.OnBreak -= OnSearBreak;
        trigger.OnDeadPull -= OnDeadPull;
    }

    // ---------- хват ----------

    void OnSelected(SelectEnterEventArgs a)
    {
        Rb.isKinematic = true;
        if (a.interactorObject is XRSocketInteractor s) { Socket = s; return; }
        var h = HandRef.Of(a.interactorObject);
        if (h == null) return;
        if (MainHand == null) MainHand = h;
        else if (SupportHand == null && h != MainHand) SupportHand = h;
        Haptics.Play(h, 0.2f, 0.03f);
    }

    void OnDeselected(SelectExitEventArgs a)
    {
        if (a.interactorObject is XRSocketInteractor s) { if (Socket == s) Socket = null; }
        else
        {
            var h = HandRef.Of(a.interactorObject);
            if (h == SupportHand) SupportHand = null;
            else if (h == MainHand) { MainHand = SupportHand; SupportHand = null; }
        }
        StartCoroutine(RestorePhysics());
    }

    IEnumerator RestorePhysics()
    {
        yield return null;                  // дать XRI закончить обработку выхода
        if (!Grab.isSelected) Rb.isKinematic = false;
    }

    // ---------- каждый кадр ----------

    void Update()
    {
        bool canFire = hammer.IsCocked && !cylinder.IsOpen;
        trigger.SetInput(MainHand != null ? MainHand.Trigger : 0f, canFire);

        // Оружие 20 секунд на полу: возвращаем в кобуру
        if (holster && !Grab.isSelected && transform.position.y < 0.3f)
        {
            _floorTimer += Time.deltaTime;
            if (_floorTimer > 20f) { _floorTimer = 0f; holster.Stow(this); }
        }
        else _floorTimer = 0f;
    }

    // ---------- события механики ----------

    void OnNotch(int n) { sfx.Notch(n, transform.position); Haptics.Play(MainHand, 0.15f, 0.012f); }
    void OnCocked() { sfx.CockLock(transform.position); Haptics.Play(MainHand, 0.3f, 0.02f); }
    void OnHammerDown() { sfx.HammerDown(transform.position); }

    void OnDeadPull()
    {
        sfx.DeadTrigger(transform.position);                // спуск без взвода: тихий «клак», выстрела нет
        Haptics.Play(MainHand, 0.1f, 0.01f);
    }

    void OnSearBreak()
    {
        Haptics.Play(MainHand, 0.25f, 0.015f);              // срыв сухаря
        if (hammer.IsHeld) { hammer.BeginDecock(); return; } // рука на курке: спуск = плавный спуск курка вниз
        hammer.Fall();
        StartCoroutine(Strike());
    }

    IEnumerator Strike()
    {
        yield return new WaitForSeconds(0.012f);            // курок падает ~12 мс
        Chamber ch = cylinder.Current;
        Cartridge cart = ch ? ch.occupant : null;
        if (cart != null && cart.state == CartState.Live) { cart.Fire(); Shoot(); }
        else
        {
            sfx.DryClick(muzzle.position);                  // пустая камора или стреляная гильза
            Haptics.Play(MainHand, 0.35f, 0.02f);
        }
    }

    void Shoot()
    {
        var b = BulletPool.Get();
        if (b) b.Launch(muzzle.position, muzzle.forward);
        sfx.Shot(muzzle.position);
        if (muzzleSmoke) muzzleSmoke.Play();
        if (gapSmoke) gapSmoke.Play();
        if (flash) StartCoroutine(Flash());
        StartCoroutine(Recoil(SupportHand != null ? twoHandRecoil : 1f));
        StartCoroutine(ShotHaptics());
        OnShot?.Invoke();
    }

    IEnumerator Flash()
    {
        flash.enabled = true;
        yield return new WaitForSeconds(0.04f);
        flash.enabled = false;
    }

    IEnumerator ShotHaptics()
    {
        Haptics.Play(MainHand, 1f, 0.045f);
        Haptics.Play(SupportHand, 0.6f, 0.045f);
        yield return new WaitForSeconds(0.045f);
        Haptics.Play(MainHand, 0.35f, 0.09f);                // хвост отдачи
        Haptics.Play(SupportHand, 0.2f, 0.09f);
    }

    // Визуальная отдача: на отдельном пивоте, трекинг руки не страдает. Камеру не трясём.
    IEnumerator Recoil(float k)
    {
        float kick = kickAngle * k, back = kickBack * k;
        for (float t = 0f; t < kickUpTime; t += Time.deltaTime) { ApplyRecoil(t / kickUpTime, kick, back); yield return null; }
        for (float t = 0f; t < kickReturnTime; t += Time.deltaTime)
        {
            ApplyRecoil(1f - Mathf.SmoothStep(0f, 1f, t / kickReturnTime), kick, back);
            yield return null;
        }
        ApplyRecoil(0f, kick, back);
    }

    void ApplyRecoil(float a, float kick, float back)
    {
        recoilPivot.localRotation = Quaternion.Euler(-kick * a, 0f, 0f);
        recoilPivot.localPosition = new Vector3(0f, 0f, -back * a);
    }
}
```

## Weapon/RevolverAudio.cs

```csharp
using UnityEngine;

// Все звуки револьвера и гильз. Пул из 10 AudioSource (3D), без аллокаций во время игры.
// Клипы: по 1–4 варианта в каждом массиве; питч ±3% для живости.
// Эхо тира: Audio Reverb Zone (Concrete/Hangar) на сцене, плюс слой «хвоста» в клипе выстрела.
// Все методы принимают необязательную позицию; без неё звук идёт от револьвера.
public class RevolverAudio : MonoBehaviour
{
    public static RevolverAudio Instance { get; private set; }

    [Header("Клипы")]
    public AudioClip[] notch;           // 4 разных щелчка трещотки (по порядку)
    public AudioClip[] cockLock;        // глухой «лок» на боевом взводе
    public AudioClip[] hammerDown;      // курок вернулся вниз без выстрела
    public AudioClip[] dryClick;        // «так» бойка по пустой камере / гильзе
    public AudioClip[] deadTrigger;     // тихий «клак» спуска без взвода
    public AudioClip[] shot;
    public AudioClip[] latch;           // защёлка барабана
    public AudioClip[] cylinderTick;    // щелчок поворота открытого барабана
    public AudioClip[] cylinderClose;   // «клац» закрытия
    public AudioClip[] ejectRod;
    public AudioClip[] seat;            // патрон встал в камору
    public AudioClip[] casingDrop;      // гильза на бетон (3–4 варианта)
    public AudioClip[] leather;         // тёрка кожи кобуры

    [Header("Громкости")]
    [Range(0f, 1f)] public float shotVolume = 1f;
    [Range(0f, 1f)] public float mechVolume = 0.7f;

    AudioSource[] _pool;
    int _next;

    void Awake()
    {
        if (Instance == null) Instance = this;
        _pool = new AudioSource[10];
        for (int i = 0; i < _pool.Length; i++)
        {
            var go = new GameObject("sfx" + i);
            go.transform.SetParent(transform, false);
            var s = go.AddComponent<AudioSource>();
            s.playOnAwake = false;
            s.spatialBlend = 1f;
            s.rolloffMode = AudioRolloffMode.Logarithmic;
            s.minDistance = 1f;
            s.maxDistance = 40f;
            _pool[i] = s;
        }
    }

    void OnDestroy() { if (Instance == this) Instance = null; }

    Vector3 P(Vector3? at) { return at ?? transform.position; }

    void Play(AudioClip[] clips, Vector3 pos, float volume, float pitchJitter = 0.03f, int index = -1)
    {
        if (clips == null || clips.Length == 0) return;
        AudioClip clip = index >= 0 ? clips[Mathf.Min(index, clips.Length - 1)] : clips[Random.Range(0, clips.Length)];
        if (!clip) return;

        AudioSource src = null;
        for (int i = 0; i < _pool.Length; i++)          // свободный источник, иначе самый старый по кругу
        {
            var c = _pool[(_next + i) % _pool.Length];
            if (!c.isPlaying) { src = c; break; }
        }
        if (src == null) { src = _pool[_next]; }
        _next = (_next + 1) % _pool.Length;

        src.transform.position = pos;
        src.pitch = 1f + Random.Range(-pitchJitter, pitchJitter);
        src.volume = volume;
        src.clip = clip;
        src.Play();
    }

    public void Notch(int n, Vector3? at = null) { Play(notch, P(at), mechVolume, 0.02f, n - 1); }
    public void CockLock(Vector3? at = null) { Play(cockLock, P(at), mechVolume); }
    public void HammerDown(Vector3? at = null) { Play(hammerDown, P(at), mechVolume * 0.8f); }
    public void DryClick(Vector3? at = null) { Play(dryClick, P(at), mechVolume); }
    public void DeadTrigger(Vector3? at = null) { Play(deadTrigger, P(at), mechVolume * 0.4f); }
    public void Shot(Vector3? at = null) { Play(shot, P(at), shotVolume); }
    public void Latch(Vector3? at = null) { Play(latch, P(at), mechVolume); }
    public void CylinderTick(Vector3? at = null) { Play(cylinderTick, P(at), mechVolume * 0.8f); }
    public void CylinderClose(Vector3? at = null) { Play(cylinderClose, P(at), mechVolume); }
    public void EjectRod(Vector3? at = null) { Play(ejectRod, P(at), mechVolume); }
    public void Seat(Vector3? at = null) { Play(seat, P(at), mechVolume * 0.8f); }

    // Громкость от скорости удара гильзы о бетон
    public void CasingDrop(Vector3 pos, float impactSpeed)
    {
        Play(casingDrop, pos, Mathf.Clamp01(impactSpeed / 3f) * 0.8f, 0.08f);
    }

    // Тёрка кожи при доставании: громкость от скорости руки
    public void LeatherRub(Vector3 pos, float handSpeed)
    {
        Play(leather, pos, Mathf.Clamp01(handSpeed / 1.5f) * 0.6f, 0.05f);
    }
}
```

## Weapon/RevolverTrigger.cs

```csharp
using System;
using UnityEngine;

// Спусковой крючок. Ось: локальная X меша. Ход 14°, срыв на 80% (≈11°), сброс на 40%.
// Без взвода спуск «мёртвый»: визуальный ход не больше 20% (≈3°), выстрела нет.
public class RevolverTrigger : MonoBehaviour
{
    public Transform mesh;
    public float maxAngle = 14f;
    public float breakAt = 0.8f;     // доля хода, на которой срывается сухарь
    public float resetAt = 0.4f;     // ниже этого спуск снова «заряжен»

    public event Action OnBreak;     // взведено и сорвали: идёт выстрел
    public event Action OnDeadPull;  // курок не взведён: сухой клик спуска

    Quaternion _rest;
    bool _armed = true;

    void Awake() { _rest = mesh.localRotation; }

    public void SetInput(float v, bool canFire)
    {
        float travel = canFire ? v : Mathf.Min(v, 0.2f);
        mesh.localRotation = _rest * Quaternion.Euler(-travel * maxAngle, 0f, 0f);

        if (_armed && v >= breakAt)
        {
            _armed = false;
            if (canFire) OnBreak?.Invoke(); else OnDeadPull?.Invoke();
        }
        else if (!_armed && v <= resetAt) _armed = true;     // чтобы стрелять снова, спуск надо отпустить
    }
}
```

## Weapon/TwoHandAim.cs

```csharp
using UnityEngine;

// Ведёт позу револьвера сам (XRI Track Position/Rotation выключены в Revolver).
// Одна рука: GripPoint = точка хвата руки, чистый трекинг без сглаживания.
// Две руки: опорная кисть тянет SupportPoint (точка на передней части рамки), вес 0.7,
// сглаживание 1 − exp(−60·dt) ≈ 17 мс. Если опорная рука ближе 4 см к рукояти (охватывает кисть),
// поворот не меняется: никакой дрожи.
// В кобуре оружие ведёт сокет: берём attach-точку сокета.
[RequireComponent(typeof(Revolver))]
public class TwoHandAim : MonoBehaviour
{
    public Transform grip;              // GripPoint, прямой ребёнок корня
    public Transform supportPoint;      // прямой ребёнок корня: ~0.07 м вперёд от рукояти, чуть ниже ствола
    [Range(0f, 1f)] public float supportWeight = 0.7f;
    public float smoothing = 60f;
    public float minSupportDistance = 0.04f;

    Revolver _gun;

    void Awake() { _gun = GetComponent<Revolver>(); }
    void OnEnable() { Application.onBeforeRender += Pose; }
    void OnDisable() { Application.onBeforeRender -= Pose; }
    void LateUpdate() { Pose(); }

    void Pose()
    {
        if (_gun == null) return;

        Transform anchor = null;
        if (_gun.MainHand != null) anchor = _gun.MainHand.interactor.GetAttachTransform(_gun.Grab);
        else if (_gun.Socket != null) anchor = _gun.Socket.GetAttachTransform(_gun.Grab);
        if (anchor == null) return;

        Vector3 gripLocalPos = grip.localPosition;
        Quaternion r = anchor.rotation * Quaternion.Inverse(grip.localRotation);   // поворот корня: GripPoint совпадает с рукой

        bool two = _gun.MainHand != null && _gun.SupportHand != null && supportPoint != null;
        if (two)
        {
            Transform sAnchor = _gun.SupportHand.interactor.GetAttachTransform(_gun.Grab);
            Vector3 from = r * (supportPoint.localPosition - gripLocalPos);   // где сейчас SupportPoint относительно рукояти
            Vector3 to = sAnchor.position - anchor.position;                  // куда его зовёт опорная рука
            if (to.magnitude > minSupportDistance && from.sqrMagnitude > 1e-6f)
            {
                Quaternion delta = Quaternion.FromToRotation(from, to);
                r = Quaternion.Slerp(Quaternion.identity, delta, supportWeight) * r;
            }
            r = Quaternion.Slerp(transform.rotation, r, 1f - Mathf.Exp(-smoothing * Time.deltaTime));
        }

        transform.rotation = r;
        transform.position = anchor.position - r * gripLocalPos;
    }
}
```

## Interaction/CylinderHandle.cs

```csharp
using UnityEngine;

// Свободная рука работает с барабаном (всё физически, без кнопок):
//  • GRIP на барабане + тянуть влево: журавль откидывается (только после нажатия защёлки);
//    чтобы закрыть, подвести барабан к раме: на ≤3° сработает фиксатор («клац»).
//  • TRIGGER (щепоть) и круговое движение руки вокруг оси барабана: открытый барабан крутится,
//    щелчок и фиксация каждые 60°.
// Объект: ребёнок SwingPivot, SphereCollider (isTrigger) R≈0.05 вокруг барабана.
// ВАЖНО: коллайдеры барабана НЕ должны входить в список colliders XRGrabInteractable револьвера,
// иначе grip на барабане превратится в обычный хват оружия.
public class CylinderHandle : HandZone
{
    public Revolver revolver;
    public Cylinder cylinder;
    public float gripStart = 0.7f, gripEnd = 0.4f;
    public float pinchStart = 0.6f, pinchEnd = 0.3f;
    public float maxDistance = 0.12f;

    HandRef _swingHand, _spinHand;
    float _swingOffset, _prevSpin;

    bool Allowed(HandRef h) { return h != revolver.MainHand && h != revolver.SupportHand; }
    bool Far(HandRef h) { return (h.PinchPos - cylinder.rotor.position).sqrMagnitude > maxDistance * maxDistance; }

    // Угол журавля, на который указывает точка мира (шарнир под барабаном, ось ‖ стволу)
    float SwingAngleOf(Vector3 world)
    {
        Transform pivot = cylinder.swingPivot;
        Vector3 p = pivot.parent.InverseTransformPoint(world) - pivot.localPosition;
        return Mathf.Atan2(-p.x, p.y) * Mathf.Rad2Deg;
    }

    // Угол руки вокруг оси ротора (против часовой вокруг +Z, как и вращение ротора)
    float SpinAngleOf(Vector3 world)
    {
        Vector3 p = cylinder.swingPivot.InverseTransformPoint(world) - cylinder.rotor.localPosition;
        return Mathf.Atan2(p.y, p.x) * Mathf.Rad2Deg;
    }

    void Update()
    {
        UpdateSwing();
        UpdateSpin();
    }

    void UpdateSwing()
    {
        if (_swingHand == null)
        {
            if (cylinder.IsLatched) return;
            foreach (var kv in inside)
            {
                HandRef h = kv.Key;
                if (!Allowed(h) || h == _spinHand || h.Grip < gripStart) continue;
                _swingHand = h;
                _swingOffset = cylinder.SwingAngle - SwingAngleOf(h.PinchPos);   // без рывка в момент хвата
                Haptics.Play(h, 0.2f, 0.02f);
                break;
            }
            return;
        }

        if (_swingHand.Grip < gripEnd || Far(_swingHand) || cylinder.IsLatched) { _swingHand = null; return; }
        cylinder.DriveSwing(SwingAngleOf(_swingHand.PinchPos) + _swingOffset);
    }

    void UpdateSpin()
    {
        if (_spinHand == null)
        {
            if (!cylinder.IsOpen) return;
            foreach (var kv in inside)
            {
                HandRef h = kv.Key;
                if (!Allowed(h) || h == _swingHand || h.Trigger < pinchStart) continue;
                _spinHand = h;
                _prevSpin = SpinAngleOf(h.PinchPos);
                break;
            }
            return;
        }

        if (_spinHand.Trigger < pinchEnd || Far(_spinHand) || !cylinder.IsOpen)
        {
            cylinder.EndSpin();
            _spinHand = null;
            return;
        }

        float a = SpinAngleOf(_spinHand.PinchPos);
        cylinder.SpinByHand(Mathf.DeltaAngle(_prevSpin, a), _spinHand);
        _prevSpin = a;
    }

    protected override void OnDisable()
    {
        base.OnDisable();
        _swingHand = null;
        _spinHand = null;
    }
}
```

## Interaction/HammerGrip.cs

```csharp
using UnityEngine;

// Взвод ВТОРОЙ рукой (основной способ): щепоть (trigger ≥ 0.6) на шпоре курка, потянуть назад по дуге.
// Объект: ребёнок Hammer.pivot, SphereCollider (isTrigger) R≈0.025 вокруг шпоры.
// Главная рука (держит револьвер) сюда не подходит: для неё есть HammerThumb.
public class HammerGrip : HandZone
{
    public Revolver revolver;
    public Hammer hammer;
    public float startTrigger = 0.6f;
    public float endTrigger = 0.3f;         // гистерезис
    public float releaseDistance = 0.10f;   // рука ушла дальше: курок отпускается

    HandRef _hand;

    void Awake() { hammer.OnNotch += OnNotch; }
    void OnDestroy() { if (hammer) hammer.OnNotch -= OnNotch; }

    void OnNotch(int n) { if (_hand != null) Haptics.Play(_hand, 0.2f, 0.012f); }   // трещотка в пальцах

    void Update()
    {
        if (_hand == null)
        {
            foreach (var kv in inside)
            {
                HandRef h = kv.Key;
                if (h == revolver.MainHand) continue;
                if (h.Trigger >= startTrigger && hammer.TryHold(this))
                {
                    _hand = h;
                    Haptics.Play(h, 0.15f, 0.02f);
                    break;
                }
            }
            return;
        }

        bool far = (_hand.PinchPos - transform.position).sqrMagnitude > releaseDistance * releaseDistance;
        if (_hand.Trigger < endTrigger || far) { hammer.Release(this); _hand = null; return; }

        hammer.Drag(this, hammer.AngleFromPoint(_hand.PinchPos));
    }

    protected override void OnDisable()
    {
        base.OnDisable();
        if (_hand != null) { hammer.Release(this); _hand = null; }
    }
}
```

## Interaction/HammerThumb.cs

```csharp
using UnityEngine;
using UnityEngine.InputSystem;

// Взвод БОЛЬШИМ пальцем главной руки. Touch-контроллеры не отслеживают большой палец,
// поэтому стик назад = курок назад (аналогово): мёртвая зона 0.25, полностью назад = 52° (взвод).
// Работает, только если главная рука доминантная (стик этой руки отдан под поворот, а не движение).
// Если стик возвращён раньше сухаря, курок падает без выстрела; спуск при зажатом курке = плавный спуск вниз.
public class HammerThumb : MonoBehaviour
{
    public Revolver revolver;
    public Hammer hammer;
    public InputActionProperty rightStick, leftStick;   // Vector2: Primary 2D Axis каждой руки
    public float deadzone = 0.25f;
    public float fullPull = 0.95f;

    bool _holding;

    void OnEnable()
    {
        rightStick.action?.Enable();
        leftStick.action?.Enable();
    }

    void Update()
    {
        HandRef h = revolver.MainHand;
        if (h == null || h.isRight != ComfortSettings.DominantRight) { Stop(); return; }

        InputAction a = (h.isRight ? rightStick : leftStick).action;
        if (a == null) { Stop(); return; }

        float pull = Mathf.InverseLerp(deadzone, fullPull, -a.ReadValue<Vector2>().y);
        if (pull <= 0f) { Stop(); return; }

        if (!_holding) _holding = hammer.TryHold(this);
        if (_holding) hammer.Drag(this, pull * Hammer.Full);
    }

    void Stop()
    {
        if (_holding) { hammer.Release(this); _holding = false; }
    }

    void OnDisable() { Stop(); }
}
```

## Interaction/HandRef.cs

```csharp
using UnityEngine;
using UnityEngine.XR.Interaction.Toolkit.Inputs.Haptics;
using UnityEngine.XR.Interaction.Toolkit.Interactors;

// Один раз на каждую руку (корень контроллера). Даёт остальным скриптам единый доступ к:
// значениям grip/trigger, точкам «щепоти» и кончика указательного, коллайдерам руки и хаптику.
// Коллайдеры руки на слое Hand: сфера на ладони R=0.04 и капсула на указательном (kinematic).
public class HandRef : MonoBehaviour
{
    [Header("Ссылки")]
    public XRBaseInputInteractor interactor;   // XRDirectInteractor этой руки
    public Transform pinchPoint;               // между подушечками большого и указательного
    public Transform indexTip;                 // кончик указательного
    public Collider[] colliders;               // если пусто, соберём из детей
    public bool isRight = true;

    HapticImpulsePlayer _haptics;

    public float Trigger => interactor ? interactor.activateInput.ReadValue() : 0f;
    public float Grip => interactor ? interactor.selectInput.ReadValue() : 0f;
    public Vector3 PinchPos => pinchPoint ? pinchPoint.position : transform.position;
    public Vector3 IndexPos => indexTip ? indexTip.position : transform.position;

    void Awake()
    {
        _haptics = GetComponentInChildren<HapticImpulsePlayer>();
        if (!_haptics) _haptics = GetComponentInParent<HapticImpulsePlayer>();
        if (colliders == null || colliders.Length == 0) colliders = GetComponentsInChildren<Collider>();
    }

    public void Haptic(float amplitude, float duration)
    {
        if (_haptics) _haptics.SendHapticImpulse(Mathf.Clamp01(amplitude * ComfortSettings.HapticScale), duration);
    }

    // Найти руку по интерактору (или по любому компоненту руки)
    public static HandRef Of(object interactorOrComponent)
    {
        var c = interactorOrComponent as Component;
        return c ? c.GetComponentInParent<HandRef>() : null;
    }
}
```

## Interaction/HandZone.cs

```csharp
using System.Collections.Generic;
using UnityEngine;

// База для зон на оружии (шпора курка, барабан): помнит, какие руки сейчас внутри триггера.
// Объект с наследником: Collider (isTrigger) на слое Weapon; у родителя есть Rigidbody (револьвер).
public abstract class HandZone : MonoBehaviour
{
    protected readonly Dictionary<HandRef, int> inside = new Dictionary<HandRef, int>();

    protected virtual void OnTriggerEnter(Collider other)
    {
        var h = HandRef.Of(other);
        if (!h) return;
        inside.TryGetValue(h, out int n);
        inside[h] = n + 1;                    // у руки несколько коллайдеров: считаем вхождения
    }

    protected virtual void OnTriggerExit(Collider other)
    {
        var h = HandRef.Of(other);
        if (!h) return;
        if (inside.TryGetValue(h, out int n))
        {
            if (n <= 1) inside.Remove(h); else inside[h] = n - 1;
        }
    }

    protected virtual void OnDisable() { inside.Clear(); }
}
```

## Interaction/PushSlider.cs

```csharp
using UnityEngine;

// Подпружиненный ползунок, который двигает кончик указательного пальца свободной руки.
// Два применения (параметр kind):
//  Latch   — защёлка барабана: travel 0.008, порог 80%, дальше срабатывает Cylinder.SetLatch.
//  Ejector — шток экстрактора: travel 0.025, порог 90%, Cylinder.PushEjector. Ребёнок SwingPivot (качается с журавлём, НЕ вращается с ротором).
// Объект: BoxCollider (isTrigger) вокруг головки ползунка. mesh — подвижная часть (может быть этим же объектом).
public class PushSlider : MonoBehaviour
{
    public enum Kind { Latch, Ejector }

    public Kind kind = Kind.Latch;
    public Cylinder cylinder;
    public Transform mesh;
    public Vector3 axis = Vector3.back;      // локальное направление хода (в сторону нажатия)
    public float travel = 0.008f;            // м
    public float returnSpeed = 0.05f;        // м/с, пружина возврата
    public float lateralRadius = 0.014f;     // палец должен попасть на ось
    public float hapticAt = 0.8f;            // доля хода для «щелчка» в пальце

    public float Value { get; private set; }

    Vector3 _rest;
    HandRef _pusher;
    float _seen;
    bool _clicked;

    void Awake()
    {
        if (!mesh) mesh = transform;
        _rest = mesh.localPosition;
        axis = axis.normalized;
    }

    void OnTriggerStay(Collider other)
    {
        var h = HandRef.Of(other);
        if (h) { _pusher = h; _seen = Time.time; }
    }

    void Update()
    {
        Transform parent = mesh.parent;
        Vector3 wAxis = parent ? parent.TransformDirection(axis).normalized : axis;
        Vector3 restWorld = parent ? parent.TransformPoint(_rest) : _rest;

        float target = 0f;
        if (_pusher != null && Time.time - _seen < 0.08f)
        {
            Vector3 d = _pusher.IndexPos - restWorld;
            float depth = Vector3.Dot(d, wAxis);
            float lateral = (d - wAxis * depth).magnitude;
            if (lateral < lateralRadius && depth > -0.015f) target = Mathf.Clamp01(depth / travel);
        }

        // вперёд за пальцем сразу, обратно с ограниченной скоростью (пружина)
        Value = target > Value ? target : Mathf.MoveTowards(Value, target, returnSpeed / travel * Time.deltaTime);
        mesh.localPosition = _rest + axis * (travel * Value);

        if (Value >= hapticAt && !_clicked) { _clicked = true; Haptics.Play(_pusher, 0.4f, 0.02f); }
        else if (Value < hapticAt * 0.5f) _clicked = false;

        if (kind == Kind.Latch) cylinder.SetLatch(Value);
        else cylinder.PushEjector(Value);
    }
}
```

## Range/AmmoTable.cs

```csharp
using UnityEngine;

// Стол с патронами. При старте высыпает 18 штук в области 55×30 см (случайный поворот по Y,
// падение с 2 см). Раз в 2 с проверяет запас: если живых патронов (не в камерах) меньше 8
// и стол вне поля зрения (> 60° от взгляда), тихо досыпает горсть: никакого «магического» появления.
public class AmmoTable : MonoBehaviour
{
    public Cartridge prefab;
    public Transform tableTop;                 // центр столешницы, Y вверх
    public Vector2 area = new Vector2(0.55f, 0.30f);
    public int count = 18;
    public int refillBelow = 8;
    public int refillBatch = 6;
    public Transform head;
    public float hiddenAngle = 60f;
    public LayerMask cartridgeMask;            // слой Cartridge: проверка пересечений при спавне
    public float spacing = 0.03f;

    float _next;

    void Start() { Spawn(count); }

    void Update()
    {
        if (Time.time < _next) return;
        _next = Time.time + 2f;

        int live = 0;
        foreach (var c in FindObjectsByType<Cartridge>(FindObjectsSortMode.None))
            if (c.state == CartState.Live && !c.seated) live++;
        if (live >= refillBelow) return;

        Vector3 to = tableTop.position - head.position;
        if (Vector3.Angle(head.forward, to) < hiddenAngle) return;      // стол в поле зрения: не трогаем

        Spawn(Mathf.Min(refillBatch, count - live));
    }

    void Spawn(int n)
    {
        int placed = 0, tries = 0;
        while (placed < n && tries++ < n * 12)
        {
            var local = new Vector3(Random.Range(-area.x * 0.5f, area.x * 0.5f), 0.02f, Random.Range(-area.y * 0.5f, area.y * 0.5f));
            Vector3 p = tableTop.TransformPoint(local);
            if (Physics.CheckSphere(p, spacing, cartridgeMask, QueryTriggerInteraction.Ignore)) continue;

            Cartridge c = Instantiate(prefab, p, tableTop.rotation * Quaternion.Euler(0f, Random.Range(0f, 360f), 0f));
            c.home = p;
            c.hasHome = true;
            placed++;
        }
    }
}
```

## Range/IShootable.cs

```csharp
using UnityEngine;

// Всё, во что можно попасть: мишень, бутылка и т.д. Bullet ищет его через GetComponentInParent.
public interface IShootable
{
    void OnHit(RaycastHit hit, Vector3 bulletVelocity);
}
```

## Range/ScoreSystem.cs

```csharp
using TMPro;
using UnityEngine;

// Счёт тира: очки, выстрелы, попадания, точность. Одна копия на сцену.
// board: TextMeshPro на табло у стрелковой позиции.
public class ScoreSystem : MonoBehaviour
{
    public static ScoreSystem Instance { get; private set; }

    public Revolver revolver;
    public Transform firingLine;             // линия огня: от неё считается дистанция до мишеней
    public TMP_Text board;

    public int Total { get; private set; }
    public int Shots { get; private set; }
    public int Hits { get; private set; }

    void Awake() { Instance = this; Refresh(); }
    void OnEnable() { if (revolver) revolver.OnShot += OnShot; }
    void OnDisable() { if (revolver) revolver.OnShot -= OnShot; }
    void OnDestroy() { if (Instance == this) Instance = null; }

    void OnShot() { Shots++; Refresh(); }

    public void Add(int points)
    {
        Hits++;
        Total += points;
        Refresh();
    }

    public void ResetScore()
    {
        Total = Shots = Hits = 0;
        Refresh();
    }

    void Refresh()
    {
        if (!board) return;
        int acc = Shots > 0 ? Mathf.RoundToInt(100f * Hits / Shots) : 0;
        board.text = "ОЧКИ  " + Total + "\nВЫСТРЕЛЫ  " + Shots + "\nТОЧНОСТЬ  " + acc + "%";
    }
}
```

## Range/Target.cs

```csharp
using System.Collections;
using System.Collections.Generic;
using UnityEngine;

// Мишень: бумажная (дырки-декали, пул 64) или стальная (откидывается на шарнире и поднимается).
// Очки: max(0, 10 − floor(r / ringWidth)) × (1 + distance / 25), r — расстояние от попадания до центра.
// Коллайдер мишени на слое Target; center.forward = нормаль мишени (в сторону стрелка).
public class Target : MonoBehaviour, IShootable
{
    public enum Kind { Paper, Steel }

    public Kind kind = Kind.Paper;
    public Transform center;
    public float ringWidth = 0.04f;
    public bool autoDistance = true;         // считать от ScoreSystem.firingLine
    public float distance = 7f;

    [Header("Бумага")]
    public GameObject decalPrefab;           // маленький чёрный квад-дырка, смотрит по -hit.normal

    [Header("Сталь")]
    public Transform hinge;                  // шарнир у нижнего края; падает вокруг локальной X
    public float fallAngle = 80f, fallTime = 0.15f, riseAfter = 3f, riseTime = 0.3f;

    [Header("Звук")]
    public AudioSource audioSrc;
    public AudioClip[] hitClips;             // «тук» для бумаги, «дзынь» для стали

    const int MaxDecals = 64;
    static readonly Queue<GameObject> Decals = new Queue<GameObject>();

    bool _down;
    Quaternion _hingeRest;

    [RuntimeInitializeOnLoadMethod(RuntimeInitializeLoadType.SubsystemRegistration)]
    static void ResetStatics() { Decals.Clear(); }

    void Start()
    {
        if (hinge) _hingeRest = hinge.localRotation;
        if (autoDistance && ScoreSystem.Instance && ScoreSystem.Instance.firingLine)
        {
            Vector3 d = center.position - ScoreSystem.Instance.firingLine.position;
            d.y = 0f;
            distance = d.magnitude;
        }
    }

    public void OnHit(RaycastHit hit, Vector3 bulletVelocity)
    {
        if (kind == Kind.Steel && _down) return;

        Vector3 d = hit.point - center.position;
        d -= Vector3.Project(d, center.forward);               // расстояние в плоскости мишени
        int ring = Mathf.Max(0, 10 - Mathf.FloorToInt(d.magnitude / ringWidth));
        int points = Mathf.RoundToInt(ring * (1f + distance / 25f));

        if (ScoreSystem.Instance) ScoreSystem.Instance.Add(points);
        PlaySound();

        if (kind == Kind.Paper) SpawnDecal(hit);
        else if (hinge) StartCoroutine(FallAndRise());
    }

    void PlaySound()
    {
        if (!audioSrc || hitClips == null || hitClips.Length == 0) return;
        audioSrc.pitch = Random.Range(0.95f, 1.05f);
        audioSrc.PlayOneShot(hitClips[Random.Range(0, hitClips.Length)]);
    }

    void SpawnDecal(RaycastHit hit)
    {
        if (!decalPrefab) return;
        GameObject d = Instantiate(decalPrefab, hit.point + hit.normal * 0.002f, Quaternion.LookRotation(-hit.normal), transform);
        Decals.Enqueue(d);
        while (Decals.Count > MaxDecals)
        {
            GameObject old = Decals.Dequeue();
            if (old) Destroy(old);
        }
    }

    IEnumerator FallAndRise()
    {
        _down = true;
        yield return Rotate(0f, fallAngle, fallTime);
        yield return new WaitForSeconds(riseAfter);
        yield return Rotate(fallAngle, 0f, riseTime);
        _down = false;
    }

    IEnumerator Rotate(float from, float to, float time)
    {
        for (float t = 0f; t < time; t += Time.deltaTime)
        {
            float a = Mathf.Lerp(from, to, Mathf.SmoothStep(0f, 1f, t / time));
            hinge.localRotation = _hingeRest * Quaternion.Euler(-a, 0f, 0f);
            yield return null;
        }
        hinge.localRotation = _hingeRest * Quaternion.Euler(-to, 0f, 0f);
    }
}
```

## Player/ComfortLocomotion.cs

```csharp
using UnityEngine;
using UnityEngine.InputSystem;

// Локомоция и поворот с упором на комфорт. Вешается на XR Origin (нужен CharacterController).
//  • Плавное движение недоминантным стиком: 1.4 м/с, разгон 6 м/с² (мягкий старт), направление по голове.
//  • Режим «ступени» (ComfortSettings.StepMove): рывок 0.25 с со скоростью 6 м/с и полная виньетка.
//  • Поворот доминантным стиком: snap-turn 30° (15/45 в настройках) или плавный 90°/с.
//  • Динамическая виньетка: FOV 110° → ~70°, вход 0.15 с, выход 0.35 с; импульс 0.1 с на snap-turn.
// Стик доминантной руки тянут назад для взвода (HammerThumb): ось Y там не используется для поворота.
[RequireComponent(typeof(CharacterController))]
public class ComfortLocomotion : MonoBehaviour
{
    public Transform head;
    public VignetteRing vignette;
    public InputActionProperty leftStick, rightStick;

    [Header("Движение")]
    public float speed = 1.4f;
    public float accel = 6f;
    public float deadzone = 0.15f;

    [Header("Ступени")]
    public float stepSpeed = 6f;
    public float stepTime = 0.25f;
    public float stepCooldown = 0.3f;

    [Header("Поворот")]
    public float smoothTurnSpeed = 90f;

    [Header("Виньетка")]
    public float fovOpen = 110f;
    public float fovMoving = 70f;
    public float fovStep = 55f;
    public float closeTime = 0.15f;
    public float openTime = 0.35f;

    CharacterController _cc;
    Vector3 _vel, _dashDir;
    float _fov = 110f, _pulse, _dashLeft, _dashCooldown;
    bool _turned;

    void Awake() { _cc = GetComponent<CharacterController>(); }

    void OnEnable()
    {
        leftStick.action?.Enable();
        rightStick.action?.Enable();
    }

    void Update()
    {
        float dt = Time.deltaTime;
        InputAction move = (ComfortSettings.DominantRight ? leftStick : rightStick).action;
        InputAction turn = (ComfortSettings.DominantRight ? rightStick : leftStick).action;

        FitController();

        float moveF = Move(move != null ? move.ReadValue<Vector2>() : Vector2.zero, dt);
        float turnF = Turn(turn != null ? turn.ReadValue<Vector2>().x : 0f, dt);

        // Целевое сужение поля зрения
        _pulse = Mathf.Max(0f, _pulse - dt);
        float k = Mathf.Max(moveF, turnF, _pulse > 0f ? 1f : 0f) * ComfortSettings.VignetteStrength;
        float open = fovOpen;
        float closed = _dashLeft > 0f ? fovStep : fovMoving;
        float target = Mathf.Lerp(open, closed, k);

        float rate = (open - closed) / (target < _fov ? closeTime : openTime);
        _fov = Mathf.MoveTowards(_fov, target, rate * dt);
        if (vignette) vignette.SetFov(_fov);
    }

    // Капсула следует за головой (комнатный масштаб)
    void FitController()
    {
        Vector3 h = transform.InverseTransformPoint(head.position);
        _cc.height = Mathf.Max(0.6f, h.y + 0.1f);
        _cc.center = new Vector3(h.x, _cc.height * 0.5f + _cc.skinWidth, h.z);
    }

    // Возвращает 0..1: насколько нужно сузить поле зрения из-за движения
    float Move(Vector2 stick, float dt)
    {
        if (stick.magnitude < deadzone) stick = Vector2.zero;
        stick = Vector2.ClampMagnitude(stick, 1f);

        Vector3 fwd = head.forward; fwd.y = 0f; fwd.Normalize();
        Vector3 right = head.right; right.y = 0f; right.Normalize();
        Vector3 dir = fwd * stick.y + right * stick.x;

        if (ComfortSettings.StepMove)
        {
            _dashCooldown -= dt;
            if (_dashLeft <= 0f && _dashCooldown <= 0f && stick.magnitude > 0.6f)
            {
                _dashLeft = stepTime;
                _dashDir = dir.normalized;
            }
            if (_dashLeft > 0f)
            {
                _dashLeft -= dt;
                _cc.Move(_dashDir * stepSpeed * dt);
                if (_dashLeft <= 0f) _dashCooldown = stepCooldown;
                return 1f;
            }
            return 0f;
        }

        _vel = Vector3.MoveTowards(_vel, dir * speed, accel * dt);
        if (_vel.sqrMagnitude > 1e-6f) _cc.Move(_vel * dt);
        return Mathf.Clamp01(_vel.magnitude / speed);
    }

    float Turn(float x, float dt)
    {
        if (ComfortSettings.SnapTurn)
        {
            if (!_turned && Mathf.Abs(x) > 0.7f)
            {
                Rotate(Mathf.Sign(x) * ComfortSettings.SnapAngle);
                _turned = true;
                _pulse = 0.1f;                       // короткий импульс виньетки
            }
            else if (Mathf.Abs(x) < 0.3f) _turned = false;
            return 0f;
        }

        if (Mathf.Abs(x) > 0.2f)
        {
            Rotate(x * smoothTurnSpeed * dt);
            return 0.6f;                              // плавный поворот сужает обзор сильнее движения
        }
        return 0f;
    }

    void Rotate(float deg) { transform.RotateAround(head.position, Vector3.up, deg); }
}
```

## Player/ComfortSettings.cs

```csharp
using UnityEngine;

// Настройки комфорта и доминантной руки. Статические значения читают остальные скрипты;
// один экземпляр на сцене загружает и сохраняет их через PlayerPrefs. UI меню вызывает Save().
public class ComfortSettings : MonoBehaviour
{
    public static float HapticScale = 1f;          // 0..1, сила вибрации
    public static float BeltOffset = 0f;           // ±0.15 м, смещение пояса/кобуры по высоте
    public static bool DominantRight = true;       // кобура и главная рука справа; движение на стике другой руки
    public static bool SnapTurn = true;            // иначе плавный поворот 90°/с
    public static float SnapAngle = 30f;           // 15 / 30 / 45
    public static float VignetteStrength = 1f;     // 0..1
    public static bool StepMove = false;           // рывковое перемещение вместо плавной локомоции
    public static bool Seated = false;             // режим «сидя»: подъём уровня пола

    void Awake() { Load(); }

    public static void Load()
    {
        HapticScale = PlayerPrefs.GetFloat("rr_haptic", 1f);
        BeltOffset = PlayerPrefs.GetFloat("rr_belt", 0f);
        DominantRight = PlayerPrefs.GetInt("rr_right", 1) == 1;
        SnapTurn = PlayerPrefs.GetInt("rr_snap", 1) == 1;
        SnapAngle = PlayerPrefs.GetFloat("rr_snapangle", 30f);
        VignetteStrength = PlayerPrefs.GetFloat("rr_vignette", 1f);
        StepMove = PlayerPrefs.GetInt("rr_step", 0) == 1;
        Seated = PlayerPrefs.GetInt("rr_seated", 0) == 1;
    }

    public static void Save()
    {
        PlayerPrefs.SetFloat("rr_haptic", HapticScale);
        PlayerPrefs.SetFloat("rr_belt", BeltOffset);
        PlayerPrefs.SetInt("rr_right", DominantRight ? 1 : 0);
        PlayerPrefs.SetInt("rr_snap", SnapTurn ? 1 : 0);
        PlayerPrefs.SetFloat("rr_snapangle", SnapAngle);
        PlayerPrefs.SetFloat("rr_vignette", VignetteStrength);
        PlayerPrefs.SetInt("rr_step", StepMove ? 1 : 0);
        PlayerPrefs.SetInt("rr_seated", Seated ? 1 : 0);
        PlayerPrefs.Save();
    }
}
```

## Player/HeightCalibration.cs

```csharp
using System.Collections;
using UnityEngine;

// Калибровка высоты. Tracking Origin Mode = Floor. Через секунду после старта запоминаем высоту глаз
// (игрок стоит прямо). Режим «сидя» (ComfortSettings.Seated) приподнимает уровень пола рига до
// референсной высоты глаз, чтобы стол, мишени и пояс оказались на привычных местах.
// Высота пояса и кобуры считается Holster от живой высоты глаз × 0.58 (+ смещение ±15 см в настройках).
public class HeightCalibration : MonoBehaviour
{
    public Transform rig;                 // XR Origin
    public Transform head;                // камера
    public float referenceEye = 1.6f;     // высота глаз, под которую сделан тир
    public float maxLift = 0.6f;

    public static float StandingEye { get; private set; } = 1.6f;

    float _baseY;

    IEnumerator Start()
    {
        _baseY = rig.position.y;
        yield return new WaitForSeconds(1f);
        Calibrate();
    }

    // Вызвать, когда игрок выпрямился (кнопка в меню или «жест»: руки вверх)
    public void Calibrate()
    {
        float eye = rig.InverseTransformPoint(head.position).y;
        if (eye > 0.4f && !ComfortSettings.Seated) StandingEye = eye;
        Apply();
    }

    public void Apply()
    {
        Vector3 p = rig.position;
        if (ComfortSettings.Seated)
        {
            float eye = head.position.y - rig.position.y;                 // с учётом текущего подъёма
            float lift = Mathf.Clamp(referenceEye - (eye - (rig.position.y - _baseY)), 0f, maxLift);
            p.y = _baseY + lift;
        }
        else p.y = _baseY;
        rig.position = p;
    }
}
```

## Player/VignetteRing.cs

```csharp
using UnityEngine;

// Виньетка комфорта без постобработки (на Quest постпроцесс дорогой): чёрное кольцо с мягким краем
// перед камерой. SetFov(110) — кольца не видно, SetFov(70) — поле зрения сужено до ~70°.
// Повесь на ребёнка камеры (Main Camera): MeshFilter + MeshRenderer, позиция (0,0,0).
// Материал: Sprites/Default (входит в Always Included Shaders), поддерживает альфу вершин.
[RequireComponent(typeof(MeshFilter), typeof(MeshRenderer))]
public class VignetteRing : MonoBehaviour
{
    public float distance = 0.3f;          // расстояние кольца от камеры, м
    public float softnessDeg = 6f;         // ширина мягкого края
    public float outerDeg = 80f;           // внешний радиус (дальше всё чёрное)
    public int segments = 48;

    Mesh _mesh;
    MeshRenderer _renderer;
    Vector3[] _verts;

    void Awake()
    {
        _renderer = GetComponent<MeshRenderer>();
        _mesh = new Mesh { name = "VignetteRing" };
        _verts = new Vector3[segments * 3];
        var colors = new Color32[segments * 3];
        var tris = new int[segments * 12];

        for (int i = 0; i < segments; i++)
        {
            int j = (i + 1) % segments;
            colors[i] = new Color32(0, 0, 0, 0);                    // внутренний край прозрачный
            colors[segments + i] = new Color32(0, 0, 0, 255);       // после мягкой зоны чёрный
            colors[2 * segments + i] = new Color32(0, 0, 0, 255);

            int a0 = i, a1 = j, b0 = segments + i, b1 = segments + j, c0 = 2 * segments + i, c1 = 2 * segments + j;
            int t = i * 12;
            tris[t] = a0; tris[t + 1] = b0; tris[t + 2] = a1;
            tris[t + 3] = a1; tris[t + 4] = b0; tris[t + 5] = b1;
            tris[t + 6] = b0; tris[t + 7] = c0; tris[t + 8] = b1;
            tris[t + 9] = b1; tris[t + 10] = c0; tris[t + 11] = c1;
        }

        _mesh.vertices = _verts;
        _mesh.colors32 = colors;
        _mesh.triangles = tris;
        GetComponent<MeshFilter>().sharedMesh = _mesh;

        var mat = new Material(Shader.Find("Sprites/Default")) { renderQueue = 4000 };
        _renderer.sharedMaterial = mat;
        _renderer.shadowCastingMode = UnityEngine.Rendering.ShadowCastingMode.Off;
        _renderer.receiveShadows = false;

        SetFov(110f);
    }

    // fovDeg — полное поле зрения «окна» (градусы)
    public void SetFov(float fovDeg)
    {
        _renderer.enabled = fovDeg < 108f;                         // при полном FOV ничего не рисуем
        if (!_renderer.enabled) return;

        float rIn = distance * Mathf.Tan(Mathf.Deg2Rad * fovDeg * 0.5f);
        float rMid = distance * Mathf.Tan(Mathf.Deg2Rad * (fovDeg * 0.5f + softnessDeg));
        float rOut = distance * Mathf.Tan(Mathf.Deg2Rad * outerDeg);

        for (int i = 0; i < segments; i++)
        {
            float a = 2f * Mathf.PI * i / segments;
            float c = Mathf.Cos(a), s = Mathf.Sin(a);
            _verts[i] = new Vector3(c * rIn, s * rIn, distance);
            _verts[segments + i] = new Vector3(c * rMid, s * rMid, distance);
            _verts[2 * segments + i] = new Vector3(c * rOut, s * rOut, distance);
        }
        _mesh.vertices = _verts;
        _mesh.bounds = new Bounds(Vector3.zero, Vector3.one * 10f);   // не отсекать по границам
    }
}
```
