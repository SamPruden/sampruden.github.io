---
layout: post
title: "Godot is not the new Unity - The anatomy of a Godot API call"
---

Like many people, I've spent the last few days looking for the new Unity. Godot has some potential, especially if it can take advantage of an influx of dev talent to drive rapid improvement. Open source is cool like that. However, one major issue holds it back - the binding layer between engine code and gameplay code is structurally built to be slow in ways which are very hard to fix without tearing everything down and rebuilding the entire API from scratch.

Godot has been used to create some successful games, so clearly this isn't always a blocker. However Unity has spent the last five years working on speeding up their scripting with crazy projects such as building two custom compilers, SIMD maths libraries, custom collections and allocators, and of course the giant (and very much unfinished) ECS project. It's been their CTO's primary focus since 2018. Clearly Unity believed that scripting performance mattered to a significant part of their userbase. Switching to Godot isn't only like going back five years in Unity - it's so much worse.

I started [a controversial but productive discussion](https://reddit.com/r/godot/comments/16j345n/is_the_c_raycasting_api_as_poor_as_it_first/) about this on the Godot subreddit a few days ago. This article is a more detailed continuation of my thoughts in that post now that I have a **little** more understanding of how Godot works. **Let's be clear here: I'm still a Godot newb, and this article *will* contain mistakes and misconceptions.**

*Note: The following contains criticisms of the Godot engine's design and engineering. Although I occasionally use some emotive language to describe my feelings about these things, the Godot developers have put in lots of hard work for the FOSS community and built something that's loved by many people, and my intent is not to offend or come across as rude to any individuals.*
## Deepdive into performing a raycast from C\#

We're going to take a deep dive into how Godot achieves the equivalent of Unity's `Physics2D.Raycast`, and what happens under the hood when we use it. To make this a little more concrete, let's start by implementing a trivial function in Unity.

### Unity
```csharp
// Simple raycast in Unity
bool GetRaycastDistanceAndNormal(Vector2 origin, Vector2 direction, out float distance, out Vector2 normal) {
    RaycastHit2D hit = Physics2D.Raycast(origin, direction);
    distance = hit.distance;
    normal = hit.normal;
    return (bool)hit;
}
```

Let's have a quick look at how this is implemented by following the calls.

```csharp
public static RaycastHit2D Raycast(Vector2 origin, Vector2 direction)
 => defaultPhysicsScene.Raycast(origin, direction, float.PositiveInfinity);

public RaycastHit2D Raycast(Vector2 origin, Vector2 direction, float distance, [DefaultValue("Physics2D.DefaultRaycastLayers")] int layerMask = -5)
{
    ContactFilter2D contactFilter = ContactFilter2D.CreateLegacyFilter(layerMask, float.NegativeInfinity, float.PositiveInfinity);
    return Raycast_Internal(this, origin, direction, distance, contactFilter);
}

[NativeMethod("Raycast_Binding")]
[StaticAccessor("PhysicsQuery2D", StaticAccessorType.DoubleColon)]
private static RaycastHit2D Raycast_Internal(PhysicsScene2D physicsScene, Vector2 origin, Vector2 direction, float distance, ContactFilter2D contactFilter)
{
    Raycast_Internal_Injected(ref physicsScene, ref origin, ref direction, distance, ref contactFilter, out var ret);
    return ret;
}

[MethodImpl(MethodImplOptions.InternalCall)]
private static extern void Raycast_Internal_Injected(
    ref PhysicsScene2D physicsScene, ref Vector2 origin, ref Vector2 direction, float distance,
    ref ContactFilter2D contactFilter, out RaycastHit2D ret);
```

Okay, so it does a tiny amount of work and efficiently shunts the call off to the unmanaged engine core via the extern mechanism. That makes sense, I'm sure Godot will do something similar. Foreshadowing.

### Godot
Let's do the same thing in Godot, [exactly as the tutorial recommends](https://docs.godotengine.org/en/stable/tutorials/physics/ray-casting.html#raycast-query).

```csharp
// Equivalent raycast in Godot
bool GetRaycastDistanceAndNormal(Vector2 origin, Vector2 direction, out float distance, out Vector2 normal)
{
    World2D world = GetWorld2D();
    PhysicsDirectSpaceState2D spaceState = world.DirectSpaceState;
    PhysicsRayQueryParameters2D queryParams = PhysicsRayQueryParameters2D.Create(origin, origin + direction);
    Godot.Collections.Dictionary hitDictionary = spaceState.IntersectRay(queryParams);

    if (hitDictionary.Count != 0)
    {
        Variant hitPositionVariant = hitDictionary[(Variant)"position"];
        Vector2 hitPosition = (Vector2)hitPositionVariant;
        Variant hitNormalVariant = hitDictionary[(Variant)"normal"];
        Vector2 hitNormal = (Vector2)hitNormalVariant;
        
        distance = (hitPosition - origin).Length();
        normal = hitNormal;
        return true;
    }

    distance = default;
    normal = default;
    return false;
}
```

The first thing that we notice is that this is longer. That is not the focus of my criticism, and is partly due to the fact that I've formatted this code verbosely in order to make it easier for us to break it down line by line. So let's do that, what's actually happening here?

We start by calling `GetWorld2D()`. In Godot, physics queries are all performed in the context of a world, and this function gets the world our code is running in. Although this `World2D` is a managed class type, this function doesn't do anything crazy like allocate every time we run it. None of these functions are going to do anything crazy like that for a simple raycast, right? Foreshadowing.

If we look inside these API calls we'll see that even ostensibly simple ones like this are implemented through some rather convoluted machinery which will have at least a little performance overhead. Let's dive into `GetWorld2D` as an example of that by unravelling some of its calls through C#. This is roughly what all the calls that return managed types look like. I've added some comments to explain what's going on.

```csharp
// This is the function we're diving into.
public World2D GetWorld2D()
{
    // MethodBind64 is a pointer to the function we're calling in C++.
    // MethodBind64 is stored in a static variable, so we have to do a memory lookup to retrieve it.
    return (World2D)NativeCalls.godot_icall_0_51(MethodBind64, GodotObject.GetPtr(this));
}

// We call into these functions which mediate API calls.
internal unsafe static GodotObject godot_icall_0_51(IntPtr method, IntPtr ptr)
{
    godot_ref godot_ref = default(godot_ref);

    // The try/finally machinery is not free. This introduces a state machine.
    // It can also block JIT optimisations.
    try
    {
        // Validation check, even though everything here is internal and should be trusted.
        if (ptr == IntPtr.Zero) throw new ArgumentNullException("ptr");

        // This calls into another function which performs the actual function pointer call
        // and puts the unmanaged result in godot_ref via a pointer.
        NativeFuncs.godotsharp_method_bind_ptrcall(method, ptr, null, &godot_ref);
        
        // This is some machinery for moving references to managed objects over the C#/C++ boundary.
        return InteropUtils.UnmanagedGetManaged(godot_ref.Reference);
    }
    finally
    {
        godot_ref.Dispose();
    }
}

// The function which actually calls the function pointer.
[global::System.Runtime.CompilerServices.MethodImpl(global::System.Runtime.CompilerServices.MethodImplOptions.AggressiveInlining)]
public static partial void godotsharp_method_bind_ptrcall( global::System.IntPtr p_method_bind,  global::System.IntPtr p_instance,  void** p_args,  void* p_ret)
{
    // But wait! 
    // _unmanagedCallbacks.godotsharp_method_bind_ptrcall is actually another
    // static variable access to retrieve another function pointer.
    _unmanagedCallbacks.godotsharp_method_bind_ptrcall(p_method_bind, p_instance, p_args, p_ret);
}

// To be honest, I haven't studied this well enough to know exactly what's happening here.
// The basic idea is straightforward - this takes a pointer to an unmanaged GodotObject,
// brings it into .Net, notifies the garbage collector of it so that it can be tracked,
// and casts it to the GodotObject type.
// Fortunately, this doesn't appear to do any allocations. Foreshadowing.
public static GodotObject UnmanagedGetManaged(IntPtr unmanaged)
{
    if (unmanaged == IntPtr.Zero) return null;

    IntPtr intPtr = NativeFuncs.godotsharp_internal_unmanaged_get_script_instance_managed(unmanaged, out var r_has_cs_script_instance);
    if (intPtr != IntPtr.Zero) return (GodotObject)GCHandle.FromIntPtr(intPtr).Target;
    if (r_has_cs_script_instance.ToBool()) return null;

    intPtr = NativeFuncs.godotsharp_internal_unmanaged_get_instance_binding_managed(unmanaged);
    object obj = ((intPtr != IntPtr.Zero) ? GCHandle.FromIntPtr(intPtr).Target : null);
    if (obj != null) return (GodotObject)obj;

    intPtr = NativeFuncs.godotsharp_internal_unmanaged_instance_binding_create_managed(unmanaged, intPtr);
    if (!(intPtr != IntPtr.Zero)) return null;

    return (GodotObject)GCHandle.FromIntPtr(intPtr).Target;
}
```

This is actually a substantial amount of overhead. We have a number of layers of pointer chasing indirection between our code and C++. Each of those is a memory lookup, and on top of that we do a bit of work with the validation, `try`  `finally`, and interpreting the returned pointer. These may sound like tiny inconsequential things, but when every single call into the core and every property/field access on a Godot object does this whole journey, it starts to add up.

If we look at the next line which accesses the `world.DirectSpaceState` property we'll find it does pretty much the same thing. The `PhysicsDirectSpaceState2D` is once again retrieved from C++ land via this machinery. Don't worry, I won't bore you with the details!

The line after that is the first thing I saw here that really boggled my bonnet.

```csharp
PhysicsRayQueryParameters2D queryParams = PhysicsRayQueryParameters2D.Create(origin, origin + direction);
```

What's the big deal, that's just a little struct packing some raycast parameters, right? **Wrong**. `PhysicsRayQueryParameters2D` is a managed class, and this is a full GC garbage generating allocation. That's a pretty crazy thing to have in a performance sensitive hot path! I'm sure it's just the one allocation though, right? Let's have a look inside.

```csharp
// Summary:
//     Returns a new, pre-configured Godot.PhysicsRayQueryParameters2D object. Use it
//     to quickly create query parameters using the most common options.
//     var query = PhysicsRayQueryParameters2D.create(global_position, global_position
//     + Vector2(0, 100))
//     var collision = get_world_2d().direct_space_state.intersect_ray(query)
public unsafe static PhysicsRayQueryParameters2D Create(Vector2 from, Vector2 to, uint collisionMask = uint.MaxValue, Array<Rid> exclude = null)
{
    // Yes, this goes through all of the same machinery discussed above.
    return (PhysicsRayQueryParameters2D)NativeCalls.godot_icall_4_731(
        MethodBind0,
        &from, &to, collisionMask,
        (godot_array)(exclude ?? new Array<Rid>()).NativeValue
    );
}
```

Uh oh. Have you spotted it yet? 

That `Array<Rid>` is a `Godot.Collections.Array`. That's another managed class type. Look what happens when we pass in a `null` value.
```csharp
(godot_array)(exclude ?? new Array<Rid>()).NativeValue
```
That's right, even if we don't pass an `exclude` array, it goes ahead and allocates a whole array on the C# heap for us anyway, just so that it can immediately convert it back into a native value representing an empty array.

In order to pass two simple `Vector2` values (16 bytes) to a raycast function, we've now done two separate garbage creating heap allocations totalling 632 bytes!

As you'll see later, we can mitigate this by caching a `PhysicsRayQueryParameters2D`. However,  as you can see from the doc comment I included above, the API clearly expects and recommends creating fresh instances for each raycast.

Let's move onto the next line. It can't get any crazier, right? Foreshadowing.

```csharp
Godot.Collections.Dictionary hitDictionary = spaceState.IntersectRay(queryParams);
```

Whelp. That shadowing wasn't very fore.

That's right, our raycast is returning an untyped dictionary. And yes, it creates garbage by allocating in on the managed heap, another 96 bytes. You have my permission to do a bemused and upset type of face now. "Oh, well maybe it at least returns null if it doesn't hit anything?" you may be thinking. No. If it doesn't hit anything, it allocates and returns an empty dictionary. 

Let's jump straight into the C++ implementation here.

```c++
Dictionary PhysicsDirectSpaceState2D::_intersect_ray(const Ref<PhysicsRayQueryParameters2D> &p_ray_query) {
    ERR_FAIL_COND_V(!p_ray_query.is_valid(), Dictionary());

    RayResult result;
    bool res = intersect_ray(p_ray_query->get_parameters(), result);

    if (!res) {
        return Dictionary();
    }

    Dictionary d;
    d["position"] = result.position;
    d["normal"] = result.normal;
    d["collider_id"] = result.collider_id;
    d["collider"] = result.collider;
    d["shape"] = result.shape;
    d["rid"] = result.rid;

    return d;
}

// This is the params struct that the inernal intersect_ray takes in.
// Nothing too crazy here (although exclude could probably be improved).
struct RayParameters {
    Vector2 from;
    Vector2 to;
    HashSet<RID> exclude;
    uint32_t collision_mask = UINT32_MAX;
    bool collide_with_bodies = true;
    bool collide_with_areas = false;
    bool hit_from_inside = false;
};

// And this is the output. A perfectly reasonable return value for a raycast.
struct RayResult {
    Vector2 position;
    Vector2 normal;
    RID rid;
    ObjectID collider_id;
    Object *collider = nullptr;
    int shape = 0;
};
```

As we can see, this is wrapping some perfectly reasonable raycast function in ungodly slow craziness. That internal `intersect_ray` is the function that should be in the API!

This C++ code allocates an untyped dictionary on the unmanaged heap. If we dig down into this dictionary, we find a hashmap as expected. It performs six hashmap lookups to initialize this dictionary (some of them may even do additional allocations, but I haven't dug that deep). But wait, this is an untyped dictionary. How does that work? Well the internal hashmap maps `Variant` to `Variant`.

Sigh. What's a `Variant`? Well the implementation is [quite complicated](https://github.com/godotengine/godot/blob/master/core/variant/variant.cpp), but in simple terms it's a big tagged union type encompassing all possible types the dictionary can hold. We can think of it as being the dynamic untyped type. What we care about is its size, which is 20 bytes.

Okay, so each of those "fields" we've written into the dictionary is now 20 bytes large. Oh, and so are the keys. Those 8 byte `Vector2` values? 20 bytes each now. That `int`? 20 bytes. You get the picture.

If we sum the sizes of the fields in `RayResult`, we're looking at 44 bytes (assuming 8 byte pointers). If we sum the sizes of the `Variant` keys and values of the dictionary, that's 2 * 6 * 20 = 240 bytes! But wait, it's a hashmap. Hashmaps don't store their data compactly, so the true size of that dictionary on the heap is at least 6x larger than the data we want to return, probably much more.

Okay, let's go back to C# and see what happens when we return this thing.

```csharp
// The function we're calling.
public Dictionary IntersectRay(PhysicsRayQueryParameters2D parameters)
{
    return NativeCalls.godot_icall_1_729(MethodBind1, GodotObject.GetPtr(this), GodotObject.GetPtr(parameters));
}

internal unsafe static Dictionary godot_icall_1_729(IntPtr method, IntPtr ptr, IntPtr arg1)
{
    godot_dictionary nativeValueToOwn = default(godot_dictionary);
    if (ptr == IntPtr.Zero) throw new ArgumentNullException("ptr");

    void** intPtr = stackalloc void*[1];
    *intPtr = &arg1;
    void** p_args = intPtr;
    NativeFuncs.godotsharp_method_bind_ptrcall(method, ptr, p_args, &nativeValueToOwn);
    return Dictionary.CreateTakingOwnershipOfDisposableValue(nativeValueToOwn);
}

internal static Dictionary CreateTakingOwnershipOfDisposableValue(godot_dictionary nativeValueToOwn)
{
    return new Dictionary(nativeValueToOwn);
}

private Dictionary(godot_dictionary nativeValueToOwn)
{
    godot_dictionary value = (nativeValueToOwn.IsAllocated ? nativeValueToOwn : NativeFuncs.godotsharp_dictionary_new());
    NativeValue = (godot_dictionary.movable)value;
    _weakReferenceToSelf = DisposablesTracker.RegisterDisposable(this);
}
```

The main things to notice here are that we're allocating a new managed (garbage creating, yada yada) dictionary in C#, and that it holds a pointer into the one created on the heap in C++. Hey, at least we're not copying the dictionary contents over! I'll take wins where I can get them at this point.

Okay, so what next?

```csharp
if (hitDictionary.Count != 0)
{
    // The cast from string to Variant can be implicit - I've made it explicit here for clarity
    Variant hitPositionVariant = hitDictionary[(Variant)"position"];
    Vector2 hitPosition = (Vector2)hitPositionVariant;
    Variant hitNormalVariant = hitDictionary[(Variant)"normal"];
    Vector2 hitNormal = (Vector2)hitNormalVariant;
    
    distance = (hitPosition - origin).Length();
    normal = hitNormal;
    return true;
}
```

Hopefully we can all follow what's happening here at this point.

If our ray didn't hit anything an empty dictionary is returned, so we check for hits by checking the count.

If we hit something, for each field we want to read we:
1. Cast `string` keys to C# `Variant` structs (This also does a call into C++)
2. Chase some more function pointers to call into C++ in the way we've come to expect by now
3. Perform a hashmap lookup to get the `Variant` holding our value (via function pointer chasing, of course)
4. Copy those 20 bytes back into C# world (yes, even though we're reading `Vector2` values which are only 8 bytes)
5. Extract the `Vector2` value from the `Variant` (Yes, it also chases pointers all the way back into C++ to do this conversion)

Well that's a lot work for returning a 44 byte struct and reading a couple of fields.

### Can we do better?
#### Caching query parameters
If you can remember as far back as `PhysicsRayQueryParameters2D`, we had the opportunity to avoid some allocations by caching, so let's do that quickly.

```csharp
readonly struct CachingRayCaster
{
    private readonly PhysicsDirectSpaceState2D spaceState;
    private readonly PhysicsRayQueryParameters2D queryParams;

    public CachingRayCaster(PhysicsDirectSpaceState2D spaceState)
    {
        this.spaceState = spaceState;
        this.queryParams = PhysicsRayQueryParameters2D.Create(Vector2.Zero, Vector2.Zero);
    }

    public bool GetDistanceAndNormal(Vector2 origin, Vector2 direction, out float distance, out Vector2 normal)
    {
        Godot.Collections.Dictionary hitDictionary = this.spaceState.IntersectRay(this.queryParams);

        if (hitDictionary.Count != 0)
        {
            Variant hitPositionVariant = hitDictionary[(Variant)"position"];
            Vector2 hitPosition = (Vector2)hitPositionVariant;
            Variant hitNormalVariant = hitDictionary[(Variant)"normal"];
            Vector2 hitNormal = (Vector2)hitNormalVariant;
            distance = (hitPosition - origin).Length();
            normal = hitNormal;
            return true;
        }

        distance = default;
        normal = default;
        return false;
    }
}
```

After the first ray, this removes 2/3rds of our per ray C#/GC allocations by count, and 632/738 of our C#/GC allocations by bytes. It's still not a good situation, but it's an improvement.

#### What about GDExtension?
As you may have heard, Godot also gives us a C++ (or Rust, or other native language) API to allow us to write high performance code. That will come to the rescue here, right? Right?

Well...

So it turns out GDExtension exposes the exact same API. Yeah. You can write fast C++ code, but you still only get an API that returns an untyped dictionary of bloated `Variant` values. It's a little better because there's no GC to worry about, but... Yeah. I recommend making another sad face right about now.

#### A whole different approach -  the `RayCast2D` node
But wait! We can take a whole different approach.

```csharp
bool GetRaycastDistanceAndNormalWithNode(RayCast2D raycastNode, Vector2 origin, Vector2 direction, out float distance, out Vector2 normal)
{
    raycastNode.Position = origin;
    raycastNode.TargetPosition = origin + direction;
    raycastNode.ForceRaycastUpdate();

    distance = (raycastNode.GetCollisionPoint() - origin).Length();
    normal = raycastNode.GetCollisionNormal();
    return raycastNode.IsColliding();
}
```

Here we have a function which takes a reference to a `RayCast2D` node in the scene. As the name suggests, this is a scene node that performs raycasts. It's implemented in C++, and it doesn't go through the same API with all of the dictionary overhead. This is a pretty clunky way to do raycasts as we need a reference to a node in the scene which we're happy to mutate, and we have to reposition the node in the scene in order to do a query, but let's take a look inside.

First we need to note that, as we've come to expect, each of these properties that we're accessing does a full pointer chasing journey into C++ land.

```csharp
public Vector2 Position
{
    get => GetPosition()
    set => SetPosition(value);
}

internal unsafe void SetPosition(Vector2 position)
{
    NativeCalls.godot_icall_1_31(MethodBind0, GodotObject.GetPtr(this), &position);
}

internal unsafe static void godot_icall_1_31(IntPtr method, IntPtr ptr, Vector2* arg1)
{
    if (ptr == IntPtr.Zero) throw new ArgumentNullException("ptr");

    void** intPtr = stackalloc void*[1];
    *intPtr = arg1;
    void** p_args = intPtr;
    NativeFuncs.godotsharp_method_bind_ptrcall(method, ptr, p_args, null);
}
```

Now let's look at what `ForceRaycastUpdate()` actually does. I'm sure you can guess the C# by now, so let's dive straight into the C++.

```c++
void RayCast2D::force_raycast_update() {
    _update_raycast_state();
}

void RayCast2D::_update_raycast_state() {
    Ref<World2D> w2d = get_world_2d();
    ERR_FAIL_COND(w2d.is_null());

    PhysicsDirectSpaceState2D *dss = PhysicsServer2D::get_singleton()->space_get_direct_state(w2d->get_space());
    ERR_FAIL_NULL(dss);

    Transform2D gt = get_global_transform();

    Vector2 to = target_position;
    if (to == Vector2()) {
        to = Vector2(0, 0.01);
    }

    PhysicsDirectSpaceState2D::RayResult rr;
    bool prev_collision_state = collided;

    PhysicsDirectSpaceState2D::RayParameters ray_params;
    ray_params.from = gt.get_origin();
    ray_params.to = gt.xform(to);
    ray_params.exclude = exclude;
    ray_params.collision_mask = collision_mask;
    ray_params.collide_with_bodies = collide_with_bodies;
    ray_params.collide_with_areas = collide_with_areas;
    ray_params.hit_from_inside = hit_from_inside;

    if (dss->intersect_ray(ray_params, rr)) {
        collided = true;
        against = rr.collider_id;
        against_rid = rr.rid;
        collision_point = rr.position;
        collision_normal = rr.normal;
        against_shape = rr.shape;
    } else {
        collided = false;
        against = ObjectID();
        against_rid = RID();
        against_shape = 0;
    }

    if (prev_collision_state != collided) {
        queue_redraw();
    }
}
```

It looks like there's a lot going on here, but it's actually quite simple. If we look carefully we can see that the structure is pretty much the same as our first `GetRaycastDistanceAndNormal` C# function. It gets the world, gets the state, builds the parameters, calls `intersect_ray` to do the actual work, then writes the result out to the properties.

But look! No heap allocations, no `Dictionary`, and no `Variant`. This is more like it! We can predict that this will be a lot faster.

### Timing it
Okay, I've made a lot of allusions to all of this overhead being dramatically problematic and we can easily see that it should be, but let's put some actual numbers to this with benchmarks.

As we've seen above, `RayCast2D.ForceRaycastUpdate()` is pretty close to a minimalist call to the physics engine's `intersect_ray`, so we can use this as a baseline. Remember that even this has some overhead from the pointer chasing function call. I've also benchmarked each of the versions of the code we've discussed. Each benchmark runs 10,000 iterations of the function under test, with warmup and outlier filtering. I disabled GC collection during the tests. I like to run my game benchmarks on weaker hardware so you may get better results if you repro, but it's the relative numbers that we care about.

Our setup is a simple scene containing a single circle collider that our ray always hits. We're interested in measuring binding overhead, not the performance of the physics engine itself. We're dealing with timings for individual rays measured in nanoseconds, so these numbers may look inconsequentially small. To better illustrate their significance, I also report "calls per frame" giving the number of times the functions could be called in in a single frame at 60fps and 120fps if the game did nothing but trivial raycasts.

| Method | Time (ns) | Baseline multiple | Per frame (60fps) | Per frame (120fps) | GC alloc (bytes) |
| --- | --- | --- | --- | --- | --- |
| `ForceRaycastUpdate` (raw engine speed, not useful) | 0.49 | 1.00 | 34,000 | 17,000 | 0 |
| `GetRaycastDistanceAndNormalWithNode` | 0.97 | 1.98 | 17,200 | 8,600 | 0 |
| `CachingRayCaster.GetDistanceAndNormal` | 7.71 | 15.73 | 2,200 | 1,100 | 96 |
| `GetRaycastDistanceAndNormal` | 24.23 | 49.45 | 688 | 344 | 728 |

Those are some significant differences!

We might expect that the fastest way to do a raycast in a reasonable engine/API is to use the function exposed for doing exactly that, [which is taught as the canonical way in the documentation](https://docs.godotengine.org/en/stable/tutorials/physics/ray-casting.html#raycast-query). As we can see, if we do that, the binding/API overhead makes this 50X slower than the raw physics engine speed. Ouch!

Using that same API but being sensible (if awkward) about caching, we can get that down to only 16X overhead. This is better, but still awful.

If our aim here is to get practical performance, we have to sidestep the proper/canonical/advertised API completely, and instead clunkily manipulate scene objects to exploit them to do our query for us. In a sensible world moving objects around in the scene and asking them to do raycasts for us would be slower than calling the raw physics API, but in fact it's 8X faster.

Even the node approach is 2X slower than the raw speed of the engine (which we're actually underestimating). This means that half of the time in that function is being spent on setting two properties and reading three properties. The binding overhead is large enough that five property accesses takes as long as a raycast. Let that sink in. *Let's not even think about the fact that in the real world we may well want to set and read even more properties, such as setting the layer mask and reading the hit collider*.

At the lower end, those numbers are actually very limiting. My current project needs more than 344 raycasts per frame, and of course it does a lot more than just raycasting. This test is a trivial scene with a single collider, if we were making the raycast do actual work in a more complex scene these numbers would be even lower! The documentation's standard way of doing raycasts would grind my whole game to a halt.

We also can't forget about the garbage creating allocations that happen in C#. I usually write games with a zero garbage per frame policy.

*Just for fun, I also benchmarked Unity. It does a full useful raycast, with parameter setting and result retrieval, in about 0.52ns. Before Godot's binding overhead, the core physics engines have comparable speed.*

## Have I cherrypicked?
When I posted the reddit thread, a number of people said that the physics API is uniquely bad and that it isn't representative of the whole engine. I certainly didn't intentionally cherrypick it - it just so happens that raycasting was the very first thing I attempted when checking out Godot. However, perhaps I'm being a little unfair, so let's examine that.

If I had wanted to cherrypick a worse method, I wouldn't have had to look far. Right next to `IntersectRay` are `IntersectPoint` and `IntersectShape`, both of which share all of the same problems as `IntersectRay` with the additional craziness that they can have multiple results, so they return a heap allocated managed `Godot.Collections.Array<Dictionary>`! Oh by the way, that `Array<T>` is actually a typed wrapper around `Godot.Collections.Array`, so every 8 byte reference to a dictionary is actually stored as a 20 byte `Variant`. Clearly I haven't picked the very worst method in the API!

If we scan the whole Godot API (via C# reflection) we luckily find that there aren't that many things which return `Dictionary`. There's an eclectic list including the `AnimationNode._GetChildNodes` method, the `Bitmap.Data` property, the `Curve2D._Data` property (and 3D), some things in `GLTFSkin`, some `TextServer` stuff, some `NavigationAgent2D` pieces, etc. None of those are great places to have slow heap allocated dictionaries, but none of them are as bad as the physics API.

However, in my experience, very few engine APIs get as much use as physics. If I look at the engine API calls in my gameplay code, they're probably 80% physics and transforms.

Let's also remember that `Dictionary` is only part of the problem. If we look a little wider for things returning `Godot.Collections.Array<T>` (remember: heap allocated, contents as `Variant`) we find lots from physics, mesh & geometry manipulation, navigation, tilemaps, rendering, and more.

Physics may be a particularly bad (but essential) area of the API, but the heap allocated type problems, as well as the general slowness of the pointer chasing, are deeply rooted throughout.

## So why are we waiting for Godot?
Godot's primary scripting language is GDScript, a dynamically typed interpreted language where almost all non primitives are heap allocated, i.e. it doesn't have a struct analogue. That sentence should have set off a cacophony of performance alarms in your head. I'll give you a moment for your ears to stop ringing.

If we look at how Godot's C++ core exposes its API we'll see something interesting.

```c++
void PhysicsDirectSpaceState3D::_bind_methods() {
    ClassDB::bind_method(D_METHOD("intersect_point", "parameters", "max_results"), &PhysicsDirectSpaceState3D::_intersect_point, DEFVAL(32));
    ClassDB::bind_method(D_METHOD("intersect_ray", "parameters"), &PhysicsDirectSpaceState3D::_intersect_ray);
    ClassDB::bind_method(D_METHOD("intersect_shape", "parameters", "max_results"), &PhysicsDirectSpaceState3D::_intersect_shape, DEFVAL(32));
    ClassDB::bind_method(D_METHOD("cast_motion", "parameters"), &PhysicsDirectSpaceState3D::_cast_motion);
    ClassDB::bind_method(D_METHOD("collide_shape", "parameters", "max_results"), &PhysicsDirectSpaceState3D::_collide_shape, DEFVAL(32));
    ClassDB::bind_method(D_METHOD("get_rest_info", "parameters"), &PhysicsDirectSpaceState3D::_get_rest_info);
}
```

This one shared mechanism is used to generate the bindings for all three scripting interfaces; GDSCript, C#, and GDExtensions. `ClassDB` collects function pointers and metadata about each of the API functions, which is then piped through various code generation systems to create the bindings for each language.

This means that every API function is designed primarily to serve the limitations of GDScript. `IntersectRay` returns an untyped dynamic `Dictionary` because GDScript doesn't have structs. Our C# and even our GDExtensions C++ code has to pay the catastrophic price for that.

This way of handling binding via function pointers also leads to significant overhead, as we've seen from simple property accesses being slow. Remember, each call first does a memory lookup to find the function pointer it wants to call, then it does another lookup to find the function pointer of a secondary function which is actually responsible for calling the function, then it calls the secondary function passing it the pointer to the primary function. All along that journey there's extra validation code, branching, and type conversions. C# (and obviously C++) has a fast mechanism for calling into native code via P/Invoke, but Godot simply doesn't use it.

**Godot has made a philosophical decision to be slow.** The only practical way to interact with the engine is via this binding layer, and its core design prevents it from ever being fast. No amount of optimising the implementation of `Dictionary` or speeding up the physics engine is going to get around the fact we're passing large heap allocated values around when we should be dealing with tiny structs. While C# and GDScript APIs remain synchronised, this will always hold the engine back.

## Okay, let's fix it then!
### What can we do without deviating from the existing binding layer?
If we assume that we still need to keep all of our APIs GDScript compatible, there are a few areas where we can probably improve things, although it won't be pretty. Let's go back to our `IntsersectRay` example.

 * `GetWorld2D().DirectStateSpace` could be compressed to one call instead of two by introducing `GetWorld2DStateSpace()`.
 * The `PhysicsRayQueryParameters2D` issues could be removed by adding an overload which takes all of the fields as parameters. This would bring us roughly inline with the `CachedRayCaster` performance (16X baseline) without having to do caching.
 * The `Dictionary` allocation could be removed by allowing us to pass in a cached/pooled dictionary to write into. This is ugly and clumsy compared to a struct, but it would remove the allocation.
 * The dictionary lookup process is still ridiculously slow. We might be able to improve on that by instead returning a class with the expected properties. The allocation here could be eliminated with the cached/pooled approach the same way it could with `Dictionary`.

These options aren't pretty or ergonomic for the user, but if we're in the business of doing ugly patches to get things running, they would probably work. This would fix the allocations but we'd still probably only be about 4X the baseline because of all of the pointer chasing across the boundary and managing of cached values. 

It may also be possible to improve the generated code for all of the pointer chasing shenanigans. I haven't studied this in detail yet, but if there are wins to find there then they'd apply to the whole API across the board, which would be cool! We could probably at least get away with removing the validation and the `try` `finally` in release builds.

### What if we're allowed to add additional APIs for C# and GDExtensions which aren't GDScript compatible?
Now we're talking! If we open up this possibility\* then in theory we could augment the `ClassDB` bindings with better ones that deal directly in structs and go through the proper P/Invoke mechanisms. This is the path to viable performance.

Unfortunately, duplicating the entire API with better versions like this would create quite a mess. There might be ways through this by marking things `[Deprecated]` and trying to guide the user in the right direction, but issues such as naming clashes would get ugly.

\* Maybe this is already possible, but I haven't found it yet. Let me know!

### What if we tear it all down and start again?
This option obviously has a lot of short term pain. Godot 4.0 has only recently happened, and now I'm talking about a backcompat breaking complete API redux like a Godot 5.0. However, if I'm honest with myself, I see this as the only viable path to the engine being in a good place in three years time. Mixing fast and slow APIs as discussed above would leave us with headaches for decades - a trap I expect the engine will probably fall into.

~~In my opinion, if Godot were to go down this route, GDScript should probably be dropped entirely. I don't really see the point of it when C# exists, and supporting it causes so much hassle. I'm clearly completely at odds with the lead Godot devs and the project philosophy on this point, so I have no expectation that this will happen. Who knows though - Unity eventually dropped UnityScript for full C#, maybe Godot will one day take the same step. Foreshadowing?~~

Edit: I'm taking the above out for now. I don't personally care about GDScript, but other people do and I don't want to take it away from them. I have no objection to C# and GDScript sitting beside each other with different APIs each optimised for the respective language's needs.

## Was the title of this article melodramatic clickbait?
Maybe a little. Not a lot.

There will be people who were making games in Unity who can make those same games in Godot without these issues mattering too much. Godot may be able to capture the lower end of Unity's market. However, Unity's recent focus on performance is a good indicator that there's demand for it. I know that I certainly care about it. Godot's performance is not just worse than Unity's, it's dramatically and systematically worse.

In some projects 95% of the CPU load is in an algorithm which never touches the engine APIs. In that case, none of this matters. (The GC always matters, but we can use GDExtensions to avoid that.) For many others, good programmatic interaction with physics/collisions and manually modifying the properties of large numbers of objects are essential to the project.

For many others, it's important to know that they can do these things if they need to. Maybe you get two years into your project thinking it will barely need raycasts at all, then you make a late game decision to add some custom CPU particles that need to be able to check collisions. It's a small aesthetic change, but suddenly you need an engine API and you're in trouble. There's a lot of talk right now about the importance of being able to trust that your engine will have your back in the future. Unity has that problem with their scummy business practices, Godot has that problem with performance.

If Godot wants to be able to capture the general Unity market (I don't actually know that it does want that) it will need to make some rapid and fundamental changes. Many of the things discussed in this article will simply not be acceptable to Unity devs. 


## Discussion
I [posted this article on the r/Godot subreddit](https://old.reddit.com/r/godot/comments/16lti15/godot_is_not_the_new_unity_the_anatomy_of_a_godot/) and there's quite an active discussion there. If you've arrived here from somewhere else and would like to give feedback or be pseudonymously rude to me on the internet, that's the place to do it. 


## Acknowledgements
 * \_Mario\_Boss on reddit for being the first to bring my attention to the `Raycast2D` node trick.
 * John Riccitiello, for finally giving me a reason to do more research on other engines.
 * Mike Bithell, for letting me steal his foreshadowing joke. I didn't actually ask permission, but he seems too nice to find me and hit me.
 * Freya Holmér, because nothing has kept me more entertained while writing this than seeing her complaining about Unreal doing physics in centimetres, and waiting until the moment she shares my horror upon discovering Godot has units like `kg pixels^2`.