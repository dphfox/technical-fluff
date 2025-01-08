---
layout: post
title: Garbage collected reference counting in Luau
image:
  path: /assets/posts/garbage-collected-reference-counting-in-luau/thumb.jpg
  width: 256
  height: 256
---
GCRC is a novel resource management technique for Luau that works around the lack of `__gc` metamethod using reference counted handles.

## A primer on the garbage collection problem

Imagine you have an object constructor like this:

```lua
local function make_person(
	name: string
)
	local self = {
		name = name,
		age = math.random(5, 16)
	}

	local function clean_up_person()
		print(`{self.name} (aged {self.age}) has been cleaned up`)
	end

	return self
end
```

We would like to run `clean_up_person` when the user of `make_person` has stopped using the returned table in all of their code.

```lua
do
	local person = make_person("Allison")
	task.wait(10)
	print("I am", person.name, "and I am", person.age)
end
-- call clean_up_person() here automatically....
```

The idiomatic way to do this would be via the `__gc` metamethod, but that's disabled in Luau for good reason. Instead, we can use a weak table to detect when the object has been garbage collected from the outside, and run our cleanup code then:

```lua
local function run_on_gc(
	object: unknown, 
	callback: () -> ()
)
	local alive_test = setmetatable({object}, {__mode = "v"})
	task.spawn(function()
		repeat task.wait(1) until alive_test[1] ~= object
	end)
end

local function make_person(
	name: string
)
	local self = {
		name = name,
		age = math.random(5, 16)
	}

	local function clean_up_person()
		print(`{self.name} (aged {self.age}) has been cleaned up`)
	end

	run_on_gc(self, clean_up_person)

	return self
end
```

The only problem is that this doesn't work, because `make_person` still contains a strong reference to `self`, which we either must change to a weak reference or set to `nil`. However, both of those options means we can't access `self.name` and `self.age` in `clean_up_person`, because the object would no longer be accessible to our code.

This has been a fundamental thorn in the side of Luau library developers for a very long time, requiring the use of manual memory management methods like `:destroy()` methods, or structured memory management frameworks like [scopes](https://fluff.blog/2023/08/30/the-next-ten-years-beyond-maids.html) or maids.

Enter: garbage collected reference counting.

The core idea of GCRC is simple; strongly store the object, and give out weak handles that reference back to the object. The handles can be destroyed freely without destroying any of the object's information. When no handles remain, run the cleanup code manually, just like any other reference counting technique.

Here's what that looks like for a simplified example with only one handle:

```lua
local function run_on_gc(
	object: unknown, 
	callback: () -> ()
)
	local alive_test = setmetatable({object}, {__mode = "v"})
	task.spawn(function()
		repeat task.wait(1) until alive_test[1] ~= object
	end)
end

local function ref_counter<T>(
	callback: () -> ()
): {}
	local handle = table.freeze {}
	run_on_gc(handle, callback)
	return handle
end

local function make_person(
	name: string
)
	local self = {
		name = name,
		age = math.random(5, 16)
	}

	local function clean_up_person()
		print(`{self.name} (aged {self.age}) has been cleaned up`)
	end

	local handle = ref_counter(clean_up_person)
	handle.value = self
	
	return handle
end
```

The user of the code will have to access the data through the handle:

```lua
do
	local person = make_person("Allison")
	task.wait(10)
	print("I am", person.value.name, "and I am", person.value.age)
end
-- clean_up_person() runs here!
```

However, they should not hold onto the data strongly themselves, because the handle would be dropped immediately:

```lua
do
	local person = make_person("Allison").value
	task.wait(10) -- clean_up_person() could run here!
	print("I am", person.name, "and I am", person.age) -- kaboom
end
```

This can be prevented by changing `make_person` to provide indirect methods rather than direct data access:

```lua
local function make_person(
	name: string
)
	local self = {
		name = name,
		age = math.random(5, 16)
	}

	local function clean_up_person()
		print(`{self.name} (aged {self.age}) has been cleaned up`)
	end

	local handle = ref_counter(clean_up_person)
	function handle.name()
		return self.name
	end
	function handle.age()
		return self.age
	end
	
	return handle
end
```

Now, users are forced to intermediate all data access with the handle, so it can't be accidentally dropped without also dropping access to the data:

```lua
do
	local person = make_person("Allison")
	task.wait(10)
	print("I am", person.name(), "and I am", person.age())
end
```

This "private data" concept maps well to local variables in closure-based OOP:

```lua
local function make_person(
	name: string
)
	local age = math.random(5, 16)

	local function clean_up_person()
		print(`{name} (aged {age}) has been cleaned up`)
	end

	local handle = ref_counter(clean_up_person)
	function handle.name()
		return name
	end
	function handle.age()
		return age
	end
	
	return handle
end
```

The technique can be generalised to support multiple handles being created at once, but that isn't done in these examples for brevity.

While GCRC makes the resource management concerns invisible to the end user, they do necessitate a polling loop, which could be undesirable and may introduce extra latency between the time an object is dropped by user code, and the time the cleanup code actually runs, potentially making it unsuitable for external IO like file locks or network request closing.