local passes, fails, undefined = 0, 0, 0

local running = 0



local function getGlobal(path)

	local value = getfenv(0)



	while value ~= nil and path ~= "" do

		local name, nextValue = string.match(path, "^([^.]+)%.?(.*)$")

		value = value[name]

		path = nextValue

	end



	return value

end



local function test(name, aliases, callback)

	running += 1



	task.spawn(function()

		if not callback then

			print("⏺️ " .. name)

		elseif not getGlobal(name) then

			fails += 1

			warn("⛔ " .. name)

		else

			local success, message = pcall(callback)

	

			if success then

				passes += 1

				print("✅ " .. name .. (message and " • " .. message or ""))

			else

				fails += 1

				warn("⛔ " .. name .. " failed: " .. message)

			end

		end

	

		local undefinedAliases = {}

	

		for _, alias in ipairs(aliases) do

			if getGlobal(alias) == nil then

				table.insert(undefinedAliases, alias)

			end

		end

	

		if #undefinedAliases > 0 then

			undefined += 1

			warn("⚠️ " .. table.concat(undefinedAliases, ", "))

		end



		running -= 1

	end)

end



-- Header and summary



print("\n")



print("UNC Environment Check")

print("✅ - Pass, ⛔ - Fail, ⏺️ - No test, ⚠️ - Missing aliases\n")



task.defer(function()

	repeat task.wait() until running == 0



	local rate = math.round(passes / (passes + fails) * 100)

	local outOf = passes .. " out of " .. (passes + fails)



	print("\n")



	print("UNC Summary")

	print("✅ Tested with a " .. rate .. "% success rate (" .. outOf .. ")")

	print("⛔ " .. fails .. " tests failed")

	print("⚠️ " .. undefined .. " globals are missing aliases")

end)



-- Cache



test("cache.invalidate", {}, function()

	local container = Instance.new("Folder")

	local part = Instance.new("Part", container)

	cache.invalidate(container:FindFirstChild("Part"))

	assert(part ~= container:FindFirstChild("Part"), "Reference `part` could not be invalidated")

end)



test("cache.iscached", {}, function()

	local part = Instance.new("Part")

	assert(cache.iscached(part), "Part should be cached")

	cache.invalidate(part)

	assert(not cache.iscached(part), "Part should not be cached")

end)



test("cache.replace", {}, function()

	local part = Instance.new("Part")

	local fire = Instance.new("Fire")

	cache.replace(part, fire)

	assert(part ~= fire, "Part was not replaced with Fire")

end)



test("cloneref", {}, function()

	local part = Instance.new("Part")

	local clone = cloneref(part)

	assert(part ~= clone, "Clone should not be equal to original")

	clone.Name = "Test"

	assert(part.Name == "Test", "Clone should have updated the original")

end)



test("compareinstances", {}, function()

	local part = Instance.new("Part")

	local clone = cloneref(part)

	assert(part ~= clone, "Clone should not be equal to original")

	assert(compareinstances(part, clone), "Clone should be equal to original when using compareinstances()")

end)



-- Closures



local function shallowEqual(t1, t2)

	if t1 == t2 then

		return true

	end



	local UNIQUE_TYPES = {

		["function"] = true,

		["table"] = true,

		["userdata"] = true,

		["thread"] = true,

	}



	for k, v in pairs(t1) do

		if UNIQUE_TYPES[type(v)] then

			if type(t2[k]) ~= type(v) then

				return false

			end

		elseif t2[k] ~= v then

			return false

		end

	end



	for k, v in pairs(t2) do

		if UNIQUE_TYPES[type(v)] then

			if type(t2[k]) ~= type(v) then

				return false

			end

		elseif t1[k] ~= v then

			return false

		end

	end



	return true

end



test("checkcaller", {}, function()

	assert(checkcaller(), "Main scope should return true")

end)



test("clonefunction", {}, function()

	local function test()

		return "success"

	end

	local copy = clonefunction(test)

	assert(test() == copy(), "The clone should return the same value as the original")

	assert(test ~= copy, "The clone should not be equal to the original")

end)



test("getcallingscript", {})



test("getscriptclosure", {"getscriptfunction"}, function()

	local module = game:GetService("CoreGui").RobloxGui.Modules.Common.Constants

	local constants = getrenv().require(module)

	local generated = getscriptclosure(module)()

	assert(constants ~= generated, "Generated module should not match the original")

	assert(shallowEqual(constants, generated), "Generated constant table should be shallow equal to the original")

end)



test("hookfunction", {"replaceclosure"}, function()

	local function test()

		return true

	end

	local ref = hookfunction(test, function()

		return false

	end)

	assert(test() == false, "Function should return false")

	assert(ref() == true, "Original function should return true")

	assert(test ~= ref, "Original function should not be same as the reference")

end)



test("iscclosure", {}, function()

	assert(iscclosure(print) == true, "Function 'print' should be a C closure")

	assert(iscclosure(function() end) == false, "Executor function should not be a C closure")

end)



test("islclosure", {}, function()

	assert(islclosure(print) == false, "Function 'print' should not be a Lua closure")

	assert(islclosure(function() end) == true, "Executor function should be a Lua closure")

end)



test("isexecutorclosure", {"checkclosure", "isourclosure"}, function()

	assert(isexecutorclosure(isexecutorclosure) == true, "Did not return true for an executor global")

	assert(isexecutorclosure(newcclosure(function() end)) == true, "Did not return true for an executor C closure")

	assert(isexecutorclosure(function() end) == true, "Did not return true for an executor Luau closure")

	assert(isexecutorclosure(print) == false, "Did not return false for a Roblox global")

end)



test("loadstring", {}, function()

	local animate = game:GetService("Players").LocalPlayer.Character.Animate

	local bytecode = getscriptbytecode(animate)

	local func = loadstring(bytecode)

	assert(type(func) ~= "function", "Luau bytecode should not be loadable!")

	assert(assert(loadstring("return ... + 1"))(1) == 2, "Failed to do simple math")

	assert(type(select(2, loadstring("f"))) == "string", "Loadstring did not return anything for a compiler error")

end)



test("newcclosure", {}, function()

	local function test()

		return true

	end

	local testC = newcclosure(test)

	assert(test() == testC(), "New C closure should return the same value as the original")

	assert(test ~= testC, "New C closure should not be same as the original")

	assert(iscclosure(testC), "New C closure should be a C closure")

end)



-- Console



test("rconsoleclear", {"consoleclear"})



test("rconsolecreate", {"consolecreate"})



test("rconsoledestroy", {"consoledestroy"})



test("rconsoleinput", {"consoleinput"})



test("rconsoleprint", {"consoleprint"})



test("rconsolesettitle", {"rconsolename", "consolesettitle"})



-- Crypt



test("crypt.base64encode", {"crypt.base64.encode", "crypt.base64_encode", "base64.encode", "base64_encode"}, function()

	assert(crypt.base64encode("test") == "dGVzdA==", "Base64 encoding failed")

end)



test("crypt.base64decode", {"crypt.base64.decode", "crypt.base64_decode", "base64.decode", "base64_decode"}, function()

	assert(crypt.base64decode("dGVzdA==") == "test", "Base64 decoding failed")

end)



test("crypt.encrypt", {}, function()

	local key = crypt.generatekey()

	local encrypted, iv = crypt.encrypt("test", key, nil, "CBC")

	assert(iv, "crypt.encrypt should return an IV")

	local decrypted = crypt.decrypt(encrypted, key, iv, "CBC")

	assert(decrypted == "test", "Failed to decrypt raw string from encrypted data")

end)



test("crypt.decrypt", {}, function()

	local key, iv = crypt.generatekey(), crypt.generatekey()

	local encrypted = crypt.encrypt("test", key, iv, "CBC")

	local decrypted = crypt.decrypt(encrypted, key, iv, "CBC")

	assert(decrypted == "test", "Failed to decrypt raw string from encrypted data")

end)



test("crypt.generatebytes", {}, function()

	local size = math.random(10, 100)

	local bytes = crypt.generatebytes(size)

	assert(#crypt.base64decode(bytes) == size, "The decoded result should be " .. size .. " bytes long (got " .. #crypt.base64decode(bytes) .. " decoded, " .. #bytes .. " raw)")

end)



test("crypt.generatekey", {}, function()

	local key = crypt.generatekey()

	assert(#crypt.base64decode(key) == 32, "Generated key should be 32 bytes long when decoded")

end)



test("crypt.hash", {}, function()

	local algorithms = {'sha1', 'sha384', 'sha512', 'md5', 'sha256', 'sha3-224', 'sha3-256', 'sha3-512'}

	for _, algorithm in ipairs(algorithms) do

		local hash = crypt.hash("test", algorithm)

		assert(hash, "crypt.hash on algorithm '" .. algorithm .. "' should return a hash")

	end

end)



--- Debug



test("debug.getconstant", {}, function()

	local function test()

		print("Hello, world!")

	end

	assert(debug.getconstant(test, 1) == "print", "First constant must be print")

	assert(debug.getconstant(test, 2) == nil, "Second constant must be nil")

	assert(debug.getconstant(test, 3) == "Hello, world!", "Third constant must be 'Hello, world!'")

end)



test("debug.getconstants", {}, function()

	local function test()

		local num = 5000 .. 50000

		print("Hello, world!", num, warn)

	end

	local constants = debug.getconstants(test)

	assert(constants[1] == 50000, "First constant must be 50000")

	assert(constants[2] == "print", "Second constant must be print")

	assert(constants[3] == nil, "Third constant must be nil")

	assert(constants[4] == "Hello, world!", "Fourth constant must be 'Hello, world!'")

	assert(constants[5] == "warn", "Fifth constant must be warn")

end)



test("debug.getinfo", {}, function()

	local types = {

		source = "string",

		short_src = "string",

		func = "function",

		what = "string",

		currentline = "number",

		name = "string",

		nups = "number",

		numparams = "number",

		is_vararg = "number",

	}

	local function test(...)

		print(...)

	end

	local info = debug.getinfo(test)

	for k, v in pairs(types) do

		assert(info[k] ~= nil, "Did not return a table with a '" .. k .. "' field")

		assert(type(info[k]) == v, "Did not return a table with " .. k .. " as a " .. v .. " (got " .. type(info[k]) .. ")")

	end

end)



test("debug.getproto", {}, function()

	local function test()

		local function proto()

			return true

		end

	end

	local proto = debug.getproto(test, 1, true)[1]

	local realproto = debug.getproto(test, 1)

	assert(proto, "Failed to get the inner function")

	assert(proto() == true, "The inner function did not return anything")

	if not realproto() then

		return "Proto return values are disabled on this executor"

	end

end)



test("debug.getprotos", {}, function()

	local function test()

		local function _1()

			return true

		end

		local function _2()

			return true

		end

		local function _3()

			return true

		end

	end

	for i in ipairs(debug.getprotos(test)) do

		local proto = debug.getproto(test, i, true)[1]

		local realproto = debug.getproto(test, i)

		assert(proto(), "Failed to get inner function " .. i)

		if not realproto() then

			return "Proto return values are disabled on this executor"

		end

	end

end)



test("debug.getstack", {}, function()

	local _ = "a" .. "b"

	assert(debug.getstack(1, 1) == "ab", "The first item in the stack should be 'ab'")

	assert(debug.getstack(1)[1] == "ab", "The first item in the stack table should be 'ab'")

end)



test("debug.getupvalue", {}, function()

	local upvalue = function() end

	local function test()

		print(upvalue)

	end

	assert(debug.getupvalue(test, 1) == upvalue, "Unexpected value returned from debug.getupvalue")

end)



test("debug.getupvalues", {}, function()

	local upvalue = function() end

	local function test()

		print(upvalue)

	end

	local upvalues = debug.getupvalues(test)

	assert(upvalues[1] == upvalue, "Unexpected value returned from debug.getupvalues")

end)



test("debug.setconstant", {}, function()

	local function test()

		return "fail"

	end

	debug.setconstant(test, 1, "success")

	assert(test() == "success", "debug.setconstant did not set the first constant")

end)



test("debug.setstack", {}, function()

	local function test()

		return "fail", debug.setstack(1, 1, "success")

	end

	assert(test() == "success", "debug.setstack did not set the first stack item")

end)



test("debug.setupvalue", {}, function()

	local function upvalue()

		return "fail"

	end

	local function test()

		return upvalue()

	end

	debug.setupvalue(test, 1, function()

		return "success"

	end)

	assert(test() == "success", "debug.setupvalue did not set the first upvalue")

end)



-- Filesystem



if isfolder and makefolder and delfolder then

	if isfolder(".tests") then

		delfolder(".tests")

	end

	makefolder(".tests")

end



test("readfile", {}, function()

	writefile(".tests/readfile.txt", "success")

	assert(readfile(".tests/readfile.txt") == "success", "Did not return the contents of the file")

end)



test("listfiles", {}, function()

	makefolder(".tests/listfiles")

	writefile(".tests/listfiles/test_1.txt", "success")

	writefile(".tests/listfiles/test_2.txt", "success")

	local files = listfiles(".tests/listfiles")

	assert(#files == 2, "Did not return the correct number of files")

	assert(isfile(files[1]), "Did not return a file path")

	assert(readfile(files[1]) == "success", "Did not return the correct files")

	makefolder(".tests/listfiles_2")

	makefolder(".tests/listfiles_2/test_1")

	makefolder(".tests/listfiles_2/test_2")

	local folders = listfiles(".tests/listfiles_2")

	assert(#folders == 2, "Did not return the correct number of folders")

	assert(isfolder(folders[1]), "Did not return a folder path")

end)



test("writefile", {}, function()

	writefile(".tests/writefile.txt", "success")

	assert(readfile(".tests/writefile.txt") == "success", "Did not write the file")

	local requiresFileExt = pcall(function()

		writefile(".tests/writefile", "success")

		assert(isfile(".tests/writefile.txt"))

	end)

	if not requiresFileExt then

		return "This executor requires a file extension in writefile"

	end

end)



test("makefolder", {}, function()

	makefolder(".tests/makefolder")

	assert(isfolder(".tests/makefolder"), "Did not create the folder")

end)



test("appendfile", {}, function()

	writefile(".tests/appendfile.txt", "su")

	appendfile(".tests/appendfile.txt", "cce")

	appendfile(".tests/appendfile.txt", "ss")

	assert(readfile(".tests/appendfile.txt") == "success", "Did not append the file")

end)



test("isfile", {}, function()

	writefile(".tests/isfile.txt", "success")

	assert(isfile(".tests/isfile.txt") == true, "Did not return true for a file")

	assert(isfile(".tests") == false, "Did not return false for a folder")

	assert(isfile(".tests/doesnotexist.exe") == false, "Did not return false for a nonexistent path (got " .. tostring(isfile(".tests/doesnotexist.exe")) .. ")")

end)



test("isfolder", {}, function()

	assert(isfolder(".tests") == true, "Did not return false for a folder")

	assert(isfolder(".tests/doesnotexist.exe") == false, "Did not return false for a nonexistent path (got " .. tostring(isfolder(".tests/doesnotexist.exe")) .. ")")

end)



test("delfolder", {}, function()

	makefolder(".tests/delfolder")

	delfolder(".tests/delfolder")

	assert(isfolder(".tests/delfolder") == false, "Failed to delete folder (isfolder = " .. tostring(isfolder(".tests/delfolder")) .. ")")

end)



test("delfile", {}, function()

	writefile(".tests/delfile.txt", "Hello, world!")

	delfile(".tests/delfile.txt")

	assert(isfile(".tests/delfile.txt") == false, "Failed to delete file (isfile = " .. tostring(isfile(".tests/delfile.txt")) .. ")")

end)



test("loadfile", {}, function()

	writefile(".tests/loadfile.txt", "return ... + 1")

	assert(assert(loadfile(".tests/loadfile.txt"))(1) == 2, "Failed to load a file with arguments")

	writefile(".tests/loadfile.txt", "f")

	local callback, err = loadfile(".tests/loadfile.txt")

	assert(err and not callback, "Did not return an error message for a compiler error")

end)



test("dofile", {})



-- Input



test("isrbxactive", {"isgameactive"}, function()

	assert(type(isrbxactive()) == "boolean", "Did not return a boolean value")

end)



test("mouse1click", {})



test("mouse1press", {})



test("mouse1release", {})



test("mouse2click", {})



test("mouse2press", {})



test("mouse2release", {})



test("mousemoveabs", {})



test("mousemoverel", {})



test("mousescroll", {})



-- Instances



test("fireclickdetector", {}, function()

	local detector = Instance.new("ClickDetector")

	fireclickdetector(detector, 50, "MouseHoverEnter")

end)



test("getcallbackvalue", {}, function()

	local bindable = Instance.new("BindableFunction")

	local function test()

	end

	bindable.OnInvoke = test

	assert(getcallbackvalue(bindable, "OnInvoke") == test, "Did not return the correct value")

end)



test("getconnections", {}, function()

	local types = {

		Enabled = "boolean",

		ForeignState = "boolean",

		LuaConnection = "boolean",

		Function = "function",

		Thread = "thread",

		Fire = "function",

		Defer = "function",

		Disconnect = "function",

		Disable = "function",

		Enable = "function",

	}

	local bindable = Instance.new("BindableEvent")

	bindable.Event:Connect(function() end)

	local connection = getconnections(bindable.Event)[1]

	for k, v in pairs(types) do

		assert(connection[k] ~= nil, "Did not return a table with a '" .. k .. "' field")

		assert(type(connection[k]) == v, "Did not return a table with " .. k .. " as a " .. v .. " (got " .. type(connection[k]) .. ")")

	end

end)



test("getcustomasset", {}, function()

	writefile(".tests/getcustomasset.txt", "success")

	local contentId = getcustomasset(".tests/getcustomasset.txt")

	assert(type(contentId) == "string", "Did not return a string")

	assert(#contentId > 0, "Returned an empty string")

	assert(string.match(contentId, "rbxasset://") == "rbxasset://", "Did not return an rbxasset url")

end)



test("gethiddenproperty", {}, function()

	local fire = Instance.new("Fire")

	local property, isHidden = gethiddenproperty(fire, "size_xml")

	assert(property == 5, "Did not return the correct value")

	assert(isHidden == true, "Did not return whether the property was hidden")

end)



test("sethiddenproperty", {}, function()

	local fire = Instance.new("Fire")

	local hidden = sethiddenproperty(fire, "size_xml", 10)

	assert(hidden, "Did not return true for the hidden property")

	assert(gethiddenproperty(fire, "size_xml") == 10, "Did not set the hidden property")

end)



test("gethui", {}, function()

	assert(typeof(gethui()) == "Instance", "Did not return an Instance")

end)



test("getinstances", {}, function()

	assert(getinstances()[1]:IsA("Instance"), "The first value is not an Instance")

end)



test("getnilinstances", {}, function()

	assert(getnilinstances()[1]:IsA("Instance"), "The first value is not an Instance")

	assert(getnilinstances()[1].Parent == nil, "The first value is not parented to nil")

end)



test("isscriptable", {}, function()

	local fire = Instance.new("Fire")

	assert(isscriptable(fire, "size_xml") == false, "Did not return false for a non-scriptable property (size_xml)")

	assert(isscriptable(fire, "Size") == true, "Did not return true for a scriptable property (Size)")

end)



test("setscriptable", {}, function()

	local fire = Instance.new("Fire")

	local wasScriptable = setscriptable(fire, "size_xml", true)

	assert(wasScriptable == false, "Did not return false for a non-scriptable property (size_xml)")

	assert(isscriptable(fire, "size_xml") == true, "Did not set the scriptable property")

	fire = Instance.new("Fire")

	assert(isscriptable(fire, "size_xml") == false, "⚠️⚠️ setscriptable persists between unique instances ⚠️⚠️")

end)



test("setrbxclipboard", {})



-- Metatable



test("getrawmetatable", {}, function()

	local metatable = { __metatable = "Locked!" }

	local object = setmetatable({}, metatable)

	assert(getrawmetatable(object) == metatable, "Did not return the meta
