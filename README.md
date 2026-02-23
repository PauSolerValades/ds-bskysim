# Heap

This is a Heap implementation for Zig. Its the same code basis as `std.PriorityQueue` from the Zig Standard Library with the following additions:
- Intrusive Index on struct for $O(1)$ access to the elements.
- Having a $O(1)$ access simplifies the removal from $O(N)$ to $O(log_2(N))$ (or $O(d log_d N)$)
- $d$-ary Heap implementation.

As I am writing this (23-02-26) this is also a rewrite of the `std.PriorityQueue` to be more aligned with the current standard for data structures, as `ArrayList`: not accepting an allocator inside of them anymore.


## O(1) Access Removal

Accessing an arbitrary element of a Heap is not optimal, it needs to go through all the list to find the element, and the inner array is not sorted. I needed for a project to access several elements very fast, so using `std.LinkedList` as inspiration, I've added some compile time checks to the struct to use an intrusive index of a struct.

```zig

const Event = struct {
    heap_index: usize = 0, // store the event position on the array
    priority: f64,

};

// comparison function for Event struct
fn lessThanEvent(context: void, a: Event, b: Event) Order {
    _ = context;
    return std.math.order(a.priority, b.priority);
}

const EHlt = Heap(Event, void, lessThanEvent);

test "intrusive behaviour" {
    var queue: EHlt = .empty;
    defer queue.deinit(ta);

    const e1 = Event{ .priority = 1 };
    const e2 = Event{ .priority = 2 };
    const e3 = Event{ .priority = 3 };
    const e4 = Event{ .priority = 4 };
    
    try queue.add(ta, e4);
    try queue.add(ta, e3);
    try queue.add(ta, e2);
    try queue.add(ta, e1);
    try expectEqual(@as(f64, 1), queue.remove().priority);
    try expectEqual(@as(f64, 2), queue.remove().priority);
    try expectEqual(@as(f64, 3), queue.remove().priority);
    try expectEqual(@as(f64, 4), queue.remove().priority);
}
```

At every element exchange in `siftUp` or `siftDown` the O(1) access with the index is going to be performed. It does not add _any_ overhead due to being a compile time check, and the if (I hope) getting optimized by the compiler.

```zig

/// checks if T is a struct. If it isnt we have to avoid
/// checking for the @hasField
const is_intrusive = blk: {
    if (@typeInfo(T) == .@"struct") {
        break :blk @hasField(T, "heap_index");
    }
    break :blk false;
};

/// Done for the intrusive event in heap.
inline fn writeItem(self: *Self, index: usize, item: T) void {
    self.items[index] = item;
    if (is_intrusive) {
        self.items[index].heap_index = index;
    }
}
```

Obviously if you just have a simple type and you want to use this, wrap the simple type into an struct.

## $O(log N)$ arbitrary Removal

In a Heap the usual operation is to pop the first element: that is a $O(log N)$ cost, access $O(1)$ because it's the first element and the swiftUp is $O(log N)$, and the intrusive index does not improve this, it was already optimal.

However, removal of an arbitrary element is $O(N)$ due to the look up of the element on the list, as it's not sorted. Here, using the intrusive access makes the removal of _any_ element of the heap $O(log N)$, as we have $O(1)$ access skipping the costly part, which is to search for the actual element to remove, bumping it down from $O(N)$ to $O(log N)$.

## d Children per Node 

`Heap` is the specific case with $d=2$ for a d-ary Heap, which is the `DaryHeap` function. This is super useful in some algorithms as Dijkstra to balance the tree, but performance wise increasing $d$ can lead to more cache hits. ((source)[https://en.wikipedia.org/wiki/D-ary_heap])

This is how you can use it:

```zig
test "add and remove same min heap multpile leafs" {
    inline for (.{ 3, 4, 8 }) |d| {
        
        const DHlt = DaryHeap(u32, d, void, lessThan);
        var queue: DHlt = .empty;
        defer queue.deinit(ta);

        try queue.add(ta, 1);
        try queue.add(ta, 1);
        try queue.add(ta, 2);
        try queue.add(ta, 2);
        try queue.add(ta, 1);
        try queue.add(ta, 1);
        try expectEqual(@as(u32, 1), queue.remove());
        try expectEqual(@as(u32, 1), queue.remove());
        try expectEqual(@as(u32, 1), queue.remove());
        try expectEqual(@as(u32, 1), queue.remove());
        try expectEqual(@as(u32, 2), queue.remove());
        try expectEqual(@as(u32, 2), queue.remove());
    }
}
```


# (Maybe) Soon:

Cool things that I don't need for my side project but they can be useful:

1. `MinMaxHeap`: Have another struct which keeps track of both the minimum and the maximum at the same time. Every difficult part is almost done, its just to combine what we already have in clever ways.
2. `MultiArrayHeap`: If the struct on the comparisons gets very big with a lot of fields, a `MultiArrayList` could be used to efficently unpack them. I have something similiar which I did months ago messing around, but it's not based on the `std`, so the quality is significantly lower.

Despite that, if you need to use a Heap with a struct with lots of field you can always do:

```zig
const BigStruct = struct { a: lot, of: fileds, dude: really };

const BigReference = struct { priority: f64, data: *BigStruct };
```

I would have to think about this, but that's a way to get out of the hole.
