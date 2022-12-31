# MIRROR

This is a mirror! [Click here for the original](https://gist.github.com/TheGreatSageEqualToHeaven/e0e1dc2698307c93f6013b9825705899)

A warning to Roblox developers about a powerful exploit primitive. In this, I will detail the research I’ve conducted into this attack vector and walk you through how you as a developer, can protect against exploits with primitives like this.

DataStoreService lets you store data that needs to persist between sessions, such as items in a player’s inventory or skill points. Data stores are consistent per experience, so any place in an experience can access and change the same data, including places on different servers.

By default, experiences tested in Studio cannot access data stores, so you must first enable API services. You will need to do this to test the vulnerabilities.

The idea I wanted to explore when pondering the above question was; can we exploit remotes to prevent data from saving? It is easy to blame the developer for not protecting themselves against such a simple exploit but it ends up being more complicated than that. I found plenty of examples of these vulnerabilities occurring in many popular games such as "Adopt Me!", "Jailbreak", "RoCitizens" and many other games. 

The reason such an exploit becomes extremely powerful is because of different types of a game's economy design. Some games may not be affected by such a vulnerability and other games might. An RPG could have a boss that you can only fight once per day per player and it could have a chance to drop a rare in-game item, the vulnerability would allow an exploiter to constantly rollback their data and attempt to get the item again. A great example is "Adopt Me!" where the game has a very stable economy due to the effort put in place to prevent exploits, the game economy could become compromised due to a wide-spread duplication exploit using a rollback.

##### (All code examples are pseudo-code)

## - SetAsync & UpdateAsync

We can use Roblox Studio to test a number of interesting vulnerabilities on a Data store and look for results.

The most simple and easy to patch method of abusing this primitive is using an Instance or userdata and causing it to rollback the player's data. When buying an item the player could choose a color from a color-picker, the developer would add this item with its color to the player's data without thinking too much of it. An exploiter would be able to abuse the remote to add an Instance to the player's data and when the game would attempt to save the Data store would throw an error preventing the game from saving data.

#### Example: 
```lua
BuyItem:InvokeServer("Painting", {
	["Name"] = "Mona Lisa",
	["Color"] = workspace --// Was originally Color3.fromRgb(x,y,z)
})
```
#### Example 2: 
```lua
UpdateCanvas:FireServer(
	"Image", 
	workspace --// Originally a number
)
```

#### Patch:
```lua
if type(input) ~= "type you need here" then 
   return
end
--// Remote code below
```

Another very simple but harder to patch vulnerability is the use of the bytes from 128 to 255, all of these bytes will throw an error if passed to a Data store. This is not an issue if developers properly blacklist any vulnerable characters that could cause an issue. The simplicity of these first two exploits makes them very powerful because they are very common and efficient. 

#### Example: 
```lua
ChangePetName:FireServer({
	PetId = 1,
	PetName = "\255" --// Originally a string
})
```
#### Example 2: 
```lua
ChangeBillboard:InvokeServer(workspace.MyHouse, "\255")
```

#### Patch:
```lua
if utf8.len(String) == nil then --// See https://gist.github.com/TheGreatSageEqualToHeaven/e0e1dc2698307c93f6013b9825705899?permalink_comment_id=4334757#gistcomment-4334757
    return
end
```

A lesser known but less efficient vulnerability that is easy to patch is sending a string of bytes bigger than 4mb. This vulnerability is rather uncommon because most developers have proper checks on string length to prevent any type of abuse. 

#### Example: 
```lua
ChangePosterText:FireServer(workspace.MyHouse.Poster, string.rep([[Shortened: https://paste.sh/djllCQti#ATiNGN82igXDVc81ZvWjFyz4]], 10000))
```
#### Example 2: 
```lua
SavePlaylist:FireServer({
	Playlist_Songs = {19252353,19252511,19295932},
	Playlist_Name = string.rep([[Shortened: https://paste.sh/djllCQti#ATiNGN82igXDVc81ZvWjFyz4]], 10000)
})
```

#### Patch:
```lua
if #input > 20 then 
   return
end
--// Remote code below
```

## - Obscure vulnerabilities

Some Roblox games serialize and deserialize data in different ways which can lead to weird issues. One example is changing how tables are serialized and deserialized to support ProfileService, I am not sure on why exactly this is done on some games but the issue can be exploited using NaN.

#### Example Serializer and Deserializer: 
```lua
--// Serializer
local Data = Player.Data
local ProfileData = Profile.Data
for i,v in Data do 
    ProfileData[v] = i
end

--// Deserializer
local ProfileData = Profile.Data
local Data = Player.Data
for i,v in ProfileData do 
    Data[v] = i
end 
```

We can abuse this simple serializer by making an unprotected settings remote send 0/0 which will cause an error when serializing. A similar vulnerability can be caused by passing nil instead but that is extremely uncommon.

#### Example: 
```lua
ChangeSetting:FireServer("ViewDistance", 0/0)
```

#### Patch:
```lua
if n ~= n then 
    return
end
--// Remote code below
```

A much more dangerous vulnerability can occur when using JSON. An exploiter could attempt a JSON injection attack, although less common and dangerous than an SQL injection. I wont be covering JSON injection attacks in this but you can read an article on them.

#### Article: https://www.comparitech.com/net-admin/json-injection-guide/

An extremely uncommon vulnerability that I had only encountered in one game was setting data to nil so UpdateAsync would cancel the write operation. I doubt this will ever happen in any game that stores more than one string of data.

#### Example: 
```lua
ChangeSkill:FireServer(nil)
```

#### Patch:

```lua
if input == nil then   
    return
end
--// Remote code below
```

## - SAMPLES 
```lua
print("\\255", pcall(function()
	game:GetService("DataStoreService"):GetDataStore("Test"):SetAsync("Test", "\255")
end))

print("Instance", pcall(function()
	game:GetService("DataStoreService"):GetDataStore("Test"):SetAsync("Test", workspace)
end))

print("4mb limit", pcall(function()
	game:GetService("DataStoreService"):GetDataStore("Test"):SetAsync("Test", string.rep([[
Ǆ

؁

‱

ஹ

௸

௵

꧄

.

ဪ

꧅

⸻

𒈙

𒐫

﷽


𒌄

𒈟

𒍼

𒁎

𒀱

𒌧

𒅃 𒈓

𒍙

𒊎

𒄡

𒅌

𒁏

𒀰

𒐪

𒐩

𒈙

𒐫


𱁬 84

𰽔 76

𪚥 64

䨻 52

龘 48

䲜 44

       á́́́́́́́́́́́́́́́́́́́́́́́́́́́́́
 ̺̺̺̺̺̺̺̺̺̺̺̺̺̺̺̺̺̺̺̺̺̺̺̺̺̺̺̺̺̺̺̺̺̺̺̺̺̺̺̺̺̺̺̺̺̺̺̺̺̺̺̺̺̺̺̺̺̺̺̺̺̺̺̺̺̺̺̺̺ͩͩͩͩͩͩͩͩͩͩͩͩͩͩͩͩͩͩͩͩͩͩͩͩͩͩͩͩͩͩͩͩͩͩͩͩͩͩͩͩͩͩͩͩͩͩͩͩͩ 𓀐𓂸

😃⃢👍༼;´༎ຶ ۝ ༎ຶ༽
]], 10000))
end))

print("NaN", pcall(function()
	local data = {[0/0]=true} -- this is the line that errors
	game:GetService("DataStoreService"):GetDataStore("Test"):SetAsync("Test", data)
end))

print("Nil", pcall(function()
	local data = {[nil]=true} -- this is the line that errors
	game:GetService("DataStoreService"):GetDataStore("Test"):SetAsync("Test", data)
end))

print("Update", pcall(function()
	local data = {["ExamplePlayer"] = nil}
	game:GetService("DataStoreService"):GetDataStore("Test"):UpdateAsync("Test", function()
		return data["ExamplePlayer"]
	end)
end))
```
