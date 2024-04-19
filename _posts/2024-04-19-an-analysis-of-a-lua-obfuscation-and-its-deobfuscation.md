---
layout: post
title: An Analysis of a Lua Obfuscation and its Deobfuscation
date: 2024-04-19 14:33 +0900
categories: [Programming, Reverse Engineering]
tags: [Lua, Obfuscation, Deobfuscation]
image:
  path: /assets/img/posts/lua-obs/before-decryption.png
  alt: The Lua script before decryption
---

## Headings

> Note: This is not a tutorial for obfuscation nor deobfuscation. This post is for educational purposes only.
> {: .prompt-warning }

Recently, before I serve for the military to fullfill my duty as a Korean 🇰🇷, I have some time to play around Lua obfuscation and its deobfuscation. I found a Lua obfuscation that is quite interesting and decided to analyze it. In this post, I will explain the obfuscation and its deobfuscation.

This is not a general deobfuscation that can be applied for everywhere. It is a Lua obfuscation that is used in a specific Lua script. The obfuscation is not strong, but it is enough to make the script unreadable.

## Introduction

![Lua Image](/assets/img/posts/lua-obs/lua.jpg){: width="600" height="300" }
_Lua Language_

According to Lua's about page, Lua is described as follows:

> Lua is a powerful, efficient, lightweight, embeddable scripting language. It supports procedural programming, object-oriented programming, functional programming, data-driven programming, and data description. [^footnote]

As you can see, Lua is a powerful language that can be used in various fields. It is widely used in game development or game plugins.
For instance, Roblox uses Lua as its scripting language.

However, Lua is not a perfect language. It has some weaknesses. One of the problems is that Lua is easy to **interpret**.
This means that Lua scripts can be accessed and read the original source code easily.

To prevent this, some developers obfuscate their Lua scripts. In this post, I will analyze obfuscated Lua script and deobfuscate it.

## Analysis

First, we need to gain the obfuscated Lua script. I will not provide the method how to intercept the script since this post is about the analysis and deobfuscation.

[@JoelNoguera](https://twitter.com/niemand_sec) has written a great article about how to intercept lua scripts.
You can find the article [here](https://niemand.com.ar/2019/02/15/why-leveling-if-you-can-just-use-lua-hacking-lua-inside-games/).

[@ColtonSkees](https://twitter.com/ColtonSkees) has also written a great article that explains how to intercept lua scripts.
[Here](https://colton1skees.github.io/posts/LuaReversal.html).

After gaining the obfuscated Lua script, we can start the analysis.

### Obfuscated Lua Script

This is one of the obfuscated Lua scripts that I have gained.

```lua
return function(self, player)
    return _G
    ["\u{5F}\u{006C}\u{0049}\u{0069}\u{0049}\u{049}\u{49}\u{049}\u{49}\u{049}\u{0069}\u{69}\u{006C}\u{69}\u{6C}\u{69}\u{06C}\u{49}\u{69}\u{069}\u{006C}\u{49}\u{6C}\u{6C}\u{6C}\u{49}\u{049}\u{49}\u{0049}\u{06C}\u{69}\u{69}\u{49}\u{006C}\u{006C}"]
    ["\u{69}\u{0069}\u{0049}\u{69}\u{6C}\u{49}\u{69}\u{6C}\u{6C}\u{049}\u{049}\u{049}\u{69}\u{006C}\u{0069}\u{6C}\u{6C}\u{006C}\u{6C}\u{6C}\u{6C}\u{49}\u{6C}\u{6C}\u{069}\u{49}\u{6C}\u{49}\u{49}\u{49}\u{049}\u{69}\u{49}\u{49}"]
    [(115026039) - (209161548) + (94135520)](self, player)
end
```

> I have changed the original script to prevent the script from being abused.

In the first glance, you can see that the script is obfuscated. Now, let's analyze the script.

Let's see what we can observe from the script.

1. The script returns a function.
2. The function takes two arguments, `self` and `player`.
3. The function returns `_G` object.
4. The function accesses a table with a key that is obfuscated.
5. The function calls the accessed table with arguments `self` and `player`.
6. The key is obfuscated with Unicode characters.
7. The key is obfuscated with a simple calculation.

### Restoring Basic Obfuscation

Now, we know that they are accessing a table with a key that is obfuscated. We can restore some parts.
For instance, we can restore the key that is obfuscated with Unicode characters and a simple calculation.

I wrote a simple python script.

```python
import re

def convert_lua_unicode_to_python(match):
    """Convert a Lua Unicode escape sequence to a Python-compatible Unicode character."""
    lua_sequence = match.group(0)
    # Extract the hexadecimal part of the Lua Unicode sequence.
    hex_sequence = lua_sequence[3:-1]
    # Convert the hexadecimal to an integer, then to a character.
    char = chr(int(hex_sequence, 16))
    return char

def evaluate_expression(match):
    """Evaluate the arithmetic expression found within parentheses."""
    numbers = [int(match.group(i)) for i in range(1, 6, 2)]
    operators = [match.group(i) for i in range(2, 6, 2)]

    try:
        result = eval(f"{numbers[0]} {operators[0]} {numbers[1]} {operators[1]} {numbers[2]}")
        return f"{result}"
    except Exception as e:
        print(f"Error evaluating expression '{match.group(0)}': {e}")
        return match.group(0)  # Return the original text if there's any error.

def process_lua_content(content):
    """Process the Lua content, simplifying expressions within brackets and decoding Unicode sequences."""
    # Simplify arithmetic expressions within square brackets.
    content = re.sub(r"\((\d+)\)\s*([+-])\s*\((\d+)\)\s*([+-])\s*\((\d+)\)", evaluate_expression, content)
    # Convert Lua Unicode escape sequences to actual characters.
    content = re.sub(r'\\u\{\w+\}', convert_lua_unicode_to_python, content)
    return content
```

The idea is simple.

We convert the Lua Unicode escape sequences to actual characters and evaluate the arithmetic expressions within parentheses.

Let's see the result.

```lua
return function (self,player)
return _G["_lIiIIIIIIiilililIiilIlllIIIIliiIll"]["iiIilIillIIIilillllllIlliIlIIIIiII"][11](self,player)
end
```

Now, we can see more clearly. The key is restored.

The function returns a function that accesses a table with a key `_lIiIIIIIIiilililIiilIlllIIIIliiIll` and calls the accessed table with arguments `self` and `player`.

### Investigating the Table

So, with this information, we can think since it looks like a table so we can guess that the table is a function.

Let's print what is in the table.

```lua
local table = _G["_lIiIIIIIIiilililIiilIlllIIIIliiIll"]
-- loop table
for i = 1, #table do
    local str = table[i]
    -- print the value of the table
    print(string.format('\'table[%d]\': %s', i, str))
end
```

Unfortunately, `_G["_lIiIIIIIIiilililIiilIlllIIIIliiIll"]` is not a table. It is a some kind of a game script.

When we execute the script, we get an error.

```
Error executing script: [string "..."]:4: attempt to get length of a script.lIiIIIIIIiilililIiilIlllIIIIliiIll value (local 'table')
```

What next? Are we doomed?

No, we can still analyze the script.

Without mapping `_G["_lIiIIIIIIiilililIiilIlllIIIIliiIll"]["iiIilIillIIIilillllllIlliIlIIIIiII"]`, the creator of the script cannot execute the script.
Thus, they must have a way to map the key.
As I said before, the weakness of Lua is that it is easy to interpret which means we can find the mapping function, too.

### Chunks

We need to know where do they actually map the functions.

Fourtunately, I have found them.

There are scripts called chunk which load up the internal scripts.

```lua
return function ()
return {[(1665832418)+(1405556753)-(3071389170)]="\u{005F}\u{0049}\u{74}\u{65}\u{006D}\u{073}",[(1800706488)+(1255182846)-(3055889332)]="\u{05F}\u{0049}\u{74}\u{65}\u{06D}\u{049}\u{006E}\u{66}\u{6F}\u{004D}\u{61}\u{6E}",[(345577436)+(543558389)-(889135822)]="\u{4C}\u{6F}\u{0063}\u{61}\u{6C}\u{50}\u{6C}\u{61}\u{0079}\u{65}\u{72}",[(1284327815)+(1155995015)-(2440322826)]="\u{57}\u{73}\u{55}\u{73}\u{65}\u{072}",[(1100047201)+(1811504072)-(2911551268)]="\u{043}\u{0068}\u{61}\u{0072}\u{61}\u{63}\u{74}\u{065}\u{72}\u{4E}\u{61}\u{06D}\u{065}",[(468472164)+(63565638)-(532037796)]="\u{0053}\u{65}\u{6E}\u{64}\u{065}\u{72}"}
end
```

This is a partial script of the chunk. As you can see, the chunk returns a table that maps the keys to the actual keys.

```lua
return function ()
return {[1]="_Items",[2]="_ItemInfoMan",[3]="LocalPlayer",[4]="WsUser",[5]="CharacterName"}
end
```

_This is the partial script of the chunk._

I have deobfuscated the chunk script and found that the chunk script returns a table that maps the keys to the actual keys.

```lua
return function ()
return {[(1136777751)+(1826562872)-(2963340622)]=function(__a0,__a1)local ukKKkuuKik=1908073259;ukKKkuuKik-=704505391;return (function(_oi0l1loioIil,_1Iiolo1I10II)return _1Iiolo1I10II//((954879017)+(2041865192)-(2996734209))==((538823857)+(1286045873)-(1824869621))end)(__a0,__a1);end,[(819006130)+(1980087701)-(2799093829)]=function(__a0,__a1)local lKKluikuuk=1033172464;lKKluikuuk|=634838668;return (function(_o01ilio0oI0i,_l10IiIl1lo11)return _o01ilio0oI0i[_G["\u{5F}\u{49}\u{69}\u{0049}\u{6C}\u{6C}\u{049}\u{69}\u{069}\u{049}\u{006C}\u{6C}\u{049}\u{0069}\u{0069}\u{6C}\u{6C}\u{6C}\u{69}\u{006C}\u{6C}\u{0049}\u{49}\u{69}\u{69}\u{0049}\u{49}\u{049}\u{69}\u{049}\u{0049}\u{69}\u{69}\u{69}\u{06C}\u{06C}\u{69}"]["\u{49}\u{49}\u{49}\u{49}\u{49}\u{06C}\u{006C}\u{06C}\u{069}\u{049}\u{49}\u{69}\u{0069}\u{0049}\u{49}\u{6C}\u{49}\u{6C}\u{6C}\u{06C}\u{0049}\u{049}\u{006C}\u{6C}\u{6C}\u{6C}\u{0069}\u{069}\u{6C}\u{0069}\u{69}\u{6C}"][(1820878688)+(1158726238)-(2979604921)]](_o01ilio0oI0i,_l10IiIl1lo11)~=((1952027413)+(541241365)-(2493268778))end)(__a0,__a1);end,[(1707645256)+(1096890748)-(2804536001)]=function(__a0,__a1)local KKKiKiiukk=2091496321;KKKiKiiukk|=473455857;(function(_10olil101I0l,_lIoi0ilIl1II)_G[_G["\u{5F}\u{49}\u{6C}\u{49}\u{69}\u{49}\u{069}\u{49}\u{0069}\u{69}\u{49}\u{6C}\u{0049}\u{69}\u{49}\u{49}\u{69}\u{6C}\u{49}\u{006C}\u{069}\u{49}\u{0069}\u{6C}\u{006C}\u{69}\u{69}\u{6C}\u{69}\u{69}\u{6C}\u{6C}\u{69}\u{6C}"]["\u{69}\u{69}\u{49}\u{49}\u{06C}\u{049}\u{49}\u{0049}\u{49}\u{06C}\u{49}\u{006C}\u{49}\u{69}\u{6C}\u{49}\u{6C}\u{6C}\u{049}\u{0069}\u{6C}\u{6C}\u{049}\u{6C}\u{0049}\u{49}\u{49}\u{0049}\u{6C}\u{49}\u{69}\u{69}\u{49}\u{49}"][(1434706998)+(1078627690)-(2513334679)]][_G["\u{005F}\u{006C}\u{006C}\u{69}\u{0069}\u{06C}\u{69}\u{069}\u{6C}\u{6C}\u{0069}\u{6C}\u{69}\u{49}\u{0049}\u{6C}\u{49}\u{06C}\u{49}\u{49}\u{49}\u{06C}\u{0049}\u{49}\u{006C}\u{0069}\u{069}\u{049}\u{0069}\u{49}\u{69}\u{69}\u{6C}\u{49}\u{006C}\u{49}\u{069}"]["\u{0069}\u{6C}\u{06C}\u{6C}\u{06C}\u{69}\u{0049}\u{0069}\u{49}\u{49}\u{0069}\u{69}\u{69}\u{006C}\u{6C}\u{069}\u{49}\u{69}\u{69}\u{6C}\u{49}\u{0049}\u{69}\u{06C}\u{069}\u{06C}\u{6C}\u{006C}\u{069}\u{006C}\u{006C}\u{0049}\u{049}\u{6C}\u{6C}\u{06C}\u{6C}"][(271695678)+(205103574)-(476799221)]][_lIoi0ilIl1II](_10olil101I0l)if(_lIoi0ilIl1II==((85695325)+(1081097318)-(1166792640)))then _10olil101I0l[_G["\u{005F}\u{06C}\u{6C}\u{069}\u{0069}\u{006C}\u{0069}\u{69}\u{6C}\u{6C}\u{69}\u{6C}\u{069}\u{049}\u{49}\u{6C}\u{49}\u{6C}\u{49}\u{049}\u{049}\u{6C}\u{0049}\u{0049}\u{6C}\u{0069}\u{0069}\u{49}\u{0069}\u{49}\u{069}\u{069}\u{006C}\u{49}\u{006C}\u{49}\u{069}"]["\u{069}\u{6C}\u{6C}\u{6C}\u{006C}\u{69}\u{0049}\u{69}\u{049}\u{49}\u{69}\u{0069}\u{69}\u{6C}\u{6C}\u{069}\u{49}\u{69}\u{69}\u{6C}\u{0049}\u{49}\u{69}\u{6C}\u{0069}\u{006C}\u{6C}\u{006C}\u{069}\u{6C}\u{6C}\u{0049}\u{049}\u{006C}\u{006C}\u{6C}\u{6C}"][(1146745480)+(1832623860)-(2979369308)]]=((369384419)+(126165011)-(495549430))end if(_lIoi0ilIl1II==((1021687981)+(695410795)-(1717098766)))then end end)(__a0,__a1);end,}
end
```

_This is the partial script of the chunk._

You may notice it already. Scripts called chunk are the mapping functions that map the keys to the actual keys.

```lua
return function ()
return {[1]=function(__a0,__a1)local ukKKkuuKik=1908073259;ukKKkuuKik-=704505391;return (function(_oi0l1loioIil,_1Iiolo1I10II)return _1Iiolo1I10II//(10000)==(109)end)(__a0,__a1);end,[2]=function(__a0,__a1)local lKKluikuuk=1033172464;lKKluikuuk|=634838668;return (function(_o01ilio0oI0i,_l10IiIl1lo11)return _o01ilio0oI0i[_G["_IiIllIiiIllIiilllillIIiiIIIiIIiiilli"]["IIIIIllliIIiiIIlIlllIIlllliiliil"][5]](_o01ilio0oI0i,_l10IiIl1lo11)~=(0)end)(__a0,__a1);end,[3]=function(__a0,__a1)local KKKiKiiukk=2091496321;KKKiKiiukk|=473455857;(function(_10olil101I0l,_lIoi0ilIl1II)_G[_G["_IlIiIiIiiIlIiIIilIliIilliiliillil"]["iiIIlIIIIlIlIilIllIillIlIIIIlIiiII"][9]][_G["_lliiliilliliIIlIlIIIlIIliiIiIiilIlIi"]["illlliIiIIiiilliIiilIIililllillIIllll"][31]][_lIoi0ilIl1II](_10olil101I0l)if(_lIoi0ilIl1II==(3))then _10olil101I0l[_G["_lliiliilliliIIlIlIIIlIIliiIiIiilIlIi"]["illlliIiIIiiilliIiilIIililllillIIllll"][32]]=(0)end if(_lIoi0ilIl1II==(10))then end end)(__a0,__a1);end,[4]=function(__a0,__a1)local KKkKKuulkk=1659166981;KKkKKuulkk|=206357743;return (function(_o10IiIo1ol1I,_1ioI0I1i1lii)local _i10IlIi1io1l=_G[_G["_liliIiiiillliIIlilililIIilIIllil"]["lIliiiIiIlliilIilIiIlIiIiilllIIiIiiliI"][2]][_G["_IlliiiilillliiilIilIlillliIiIIiii"]["iilIiIliiIlllIiIliliIiIIIliliilIiiilI"][1]](_G[_G["_liliIiiiillliIIlilililIIilIIllil"]["lIliiiIiIlliilIilIiIlIiIiilllIIiIiiliI"][2]],_G["_IlliiiilillliiilIilIlillliIiIIiii"]["iilIiIliiIlllIiIliliIiIIIliliilIiiilI"][2])local _Ii10o0IiIliI=_G[_G["_liliIiiiillliIIlilililIIilIIllil"]["lIliiiIiIlliilIilIiIlIiIiilllIIiIiiliI"][3]][_G["_IlliiiilillliiilIilIlillliIiIIiii"]["iilIiIliiIlllIiIliliIiIIIliliilIiiilI"][3]](_G["_IlliiiilillliiilIilIlillliIiIIiii"]["iilIiIliiIlllIiIliliIiIIIliliilIiiilI"][4],}
end
```

This is the partial script of the chunk. As you can see, the chunk returns a table that maps the keys to the actual keys.

Your question might be "How can you know that the chunk is the mapping function?"

```lua
return function (self)
_G["_lIiIIIIIIiilililIiilIlllIIIIliiIll"]["iiIilIillIIIilillllllIlliIlIIIIiII"][1](self)
end
```

```lua
return function (self,pool,entity)
_G["_lIiIIIIIIiilililIiilIlllIIIIliiIll"]["iiIilIillIIIilillllllIlliIlIIIIiII"][2](self,pool,entity)
end
```

```lua
return function (self)
_G["_lIiIIIIIIiilililIiilIlllIIIIliiIll"]["iiIilIillIIIilillllllIlliIlIIIIiII"][3](self)
end
```

You might notice that they have the same table which is `_G["_lIiIIIIIIiilililIiilIlllIIIIliiIll"]["iiIilIillIIIilillllllIlliIlIIIIiII"]` but different key `[1]`, `[2]`, `[3]`.

This is the important. It means which can create an assumption that the chunk is the mapping function.

![Structure Explanation](/assets/img/posts/lua-obs/Explain1.png)

In order words, a chunk returns a table of functions which 1st function has only one argument, 2nd function has three arguments, and 3rd function has only one argument.

Basically, the chunk is the mapping function that maps the keys to the actual keys.

### Matching the Keys

Since we have an access to the Lua environment, we can execute the chunk script and get the mapping functions.

What I did were extracted all tables of keys which are returned by the chunk script and executed the chunk script.

```python
import re
import glob

# Compile regex pattern for matching global table references
pattern = re.compile(r'_G\["_(\w+)"\]\["(\w+)"\]\[(\d+)\]')

# Initialize a dictionary to keep track of unique tables and their highest index
unique_tables = {}

# Iterate over all lua files in the "results_chunks" folder
for file_path in glob.glob(f'Decoded/*.lua'):
    # if the file name does not contain "chunk" then continue
    # This is because other files contain table of functions and not the actual data
    if "chunk" not in file_path:
        continue

    with open(file_path, 'r', encoding='utf-8') as file:
        content = file.read()
        matches = pattern.finditer(content)
        for match in matches:
            # Extract table name, sub-table name, and index
            table_name, sub_table_name, index = match.groups()
            full_table_name = f'["_{table_name}"]["{sub_table_name}"]'

            # Convert index to integer for comparison
            index = int(index)

            # Update the dictionary with the highest index found for each unique table
            if full_table_name not in unique_tables or index > unique_tables[full_table_name]:
                unique_tables[full_table_name] = index

# Convert the results to a more readable format and sort by table name
sorted_unique_tables = sorted(unique_tables.items(), key=lambda x: x[0])

luaTable = "tables = {"
for table_name, highest_index in sorted_unique_tables:
    luaTable += "{"
    luaTable += f"['_G{table_name}'] = _G{table_name}"
    luaTable += "},"

luaTable += "}"
```

`luaTable` will have the Lua table that contains the all the keys with the table name.

```
_G["_IIIIiiiIlililIiIllIilliiIiiillIIliIlli"]["lliIillIIIllllilIliIIiIiIiiIlIIliIi"]':{1:"ClearAnimationTimer",2:"DisableTweenFloating",3:"Entity",4:"TransformComponent",5:"WorldPosition",6:"Position",7:"Clone",8:"SpriteEntity",9:"ZRotation",10:"SpriteRendererComponent",11:"Color",12:"MakeTween",13:"Linear",14:"a",15:"AutoDestroy",16:"SetOnEndCallback",17:"OnDestroyDrop",18:"Play",19:"isvalid",20:"_InputService",21:"Vector2",22:"_UILogic",23:"ReadBits",24:"DecompressStoreBlock",25:"DecompressFixBlock",26:"DecompressDynamicBlock",27:"result_buffer",28:"concat",29:"buffer",30:"",31:"buffer_size",32:"EffectDisplayerComponent"},
_G["_IIIIiiiliIliillIlllIiIlIIIIIilill"]["IIiiiIIiIliillililIliIlIlIilIill"]':{1:"MessageIndex",2:"Messages",3:"GetMessageByPath",4:"format",5:"%d/stop/%d",6:"QuestState",7:"answer",8:"Ask",9:"Stop",10:"SendMessage",11:"_UserService",12:"_AppService",13:"isvalid",14:"_DragDropLogic",15:"_TooltipType",16:"Entity",17:"Parent",18:"ControlTabComponent",19:"SetFocus",20:"_DataService"},
```

_This is the partial result of the Lua table._

This the actual data that is mapped by the chunk script.

Now, we have all the nessaary information to deobfuscate the script.

We know what the keys are and we know how to access the keys.

### Deobfuscation

By using the previous information, we can deobfuscate the inside of each function.

![Before Decryption](/assets/img/posts/lua-obs/before-decryption.png)
_Before Decryption_

![After Decryption](/assets/img/posts/lua-obs/after-decryption.png)
_After Decryption_

What I did was to replace the encrypted keys such as `["_ililIIIlIiillIlIlllIlIllIlliIiii"]["IIlilIIIIllliIilIIiilIIlliIIiIIiil"][3]` to actual keys such as `["Entity"]["Parent"]`.

Now, we can match up encrypted scripts by using the number of arguments and unique keys.

```python
import os
import re

def find_functions_in_file(file_path):
    """Finds functions and their arguments in a Lua file."""
    with open(file_path, 'r', encoding='utf-8') as file:
        content = file.read()
    return re.findall(r'\[(\d+)\]=function\((.*?)\)', content)

def find_functions_in_files(directory):
    """Finds functions and their arguments in all Lua files within the given directory."""
    functions_in_files = {}
    for root, dirs, files in os.walk(directory):
        for file in files:
            if file.endswith('.lua'):
                file_path = os.path.join(root, file)
                with open(file_path, 'r', encoding='utf-8') as lua_file:
                    content = lua_file.read()
                    functions = re.findall(r'\[(\d+)\]=function\((.*?)\)', content)
                    if functions:
                        functions_in_files[os.path.basename(file_path)] = functions
    return functions_in_files

def find_global_table_function_calls(file_path):
    """Finds global table function calls within a Lua file."""
    pattern = re.compile(r'_G\["([^"]+?)"\]\["([^"]+?)"\]\[(\d+)\]\((.*?)\)')
    with open(file_path, 'r', encoding='utf-8') as file:
        content = file.read()
    return pattern.findall(content)

def categorize_by_global_table(files):
    """
    Categorizes the given files by the global table they use.
    Returns a dictionary where each key is a global table and its value is a list of tuples (filename, function index, arguments).
    """
    global_table_usage = {}

    global_call_pattern = re.compile(r'_G\["([^"]+?)"\]\["([^"]+?)"\]\[(\d+)\]\((.*?)\)')

    for file_path in files:
        with open(file_path, 'r', encoding='utf-8') as file:
            content = file.read()
            matches = global_call_pattern.findall(content)
            for match in matches:
                table_name, sub_table, func_index, args = match
                key = f'{table_name}.{sub_table}'
                if key not in global_table_usage:
                    global_table_usage[key] = []
                global_table_usage[key].append((file_path.split('/')[-1], func_index, args))

    return global_table_usage

# Function to extract and return all function bodies from the Lua file content
def extract_functions_from_lua(lua_content):
    # This regex pattern is designed to match function bodies in the Lua content.
    # It looks for patterns starting with '[number]=function' and captures until the corresponding 'end' keyword.
    # Due to the complexity and variations in Lua syntax, this pattern might not capture all nuances perfectly.
    pattern = re.compile(r'\[(\d+)\]=function\((.*?)\)(.*?)(?=\(__a\d+(?:,__a\d+)*\);end)', re.DOTALL)

    functions = pattern.findall(lua_content)
    return functions

# Function to read and return the contents of a specified file within the "results_chunks" folder
def read_file_content(folderPath, file_name):
    file_path = os.path.join(folderPath, file_name)
    if os.path.exists(file_path):
        with open(file_path, 'r', encoding='utf-8') as file:
            return file.read()
    else:
        return "File does not exist."


def rename_arguments(lua_function, new_argument_names):
    # Split the Lua function into lines
    lines = lua_function.split('\n')

    # Regex pattern to match Lua function arguments
    arg_pattern = re.compile(r'\bfunction\b\s*\((.*?)\)\s*')

    # Find the arguments in the Lua function
    match = arg_pattern.search(lines[0])
    if match:
        old_arguments = match.group(1).split(',')
    else:
        return "Function arguments not found"

    # Check if the number of old and new arguments match
    if len(old_arguments) != len(new_argument_names):
        return "Number of new argument names does not match the number of old arguments"

    # Rename the arguments in the Lua function
    new_lines = []
    for line in lines:
        for old_arg, new_arg in zip(old_arguments, new_argument_names):
            line = re.sub(r'\b' + re.escape(old_arg.strip()) + r'\b', new_arg.strip(), line)
        new_lines.append(line)

    return '\n'.join(new_lines)

def rename_locals(lua_function):
    # Split the Lua function into lines
    lines = lua_function.split('\n')

    # Regex pattern to match local variable declarations
    local_pattern = re.compile(r'\blocal\b\s*(\w+)\s*')

    # Counter for generating unique local variable names
    local_counter = 1

    # Rename local variables in the Lua function
    new_lines = []
    for line in lines:
        match = local_pattern.search(line)
        if match:
            old_local = match.group(1)
            new_local = f'local_{local_counter}'
            line = re.sub(r'\b' + re.escape(old_local) + r'\b', new_local, line)
            local_counter += 1
        new_lines.append(line)

    return '\n'.join(new_lines)

# Compile regex pattern for matching global table references
localValPattern = re.compile(r'local\s+(_\w+)(?:\s*=\s*\w+)?(?:\s*,\s*(_\w+)(?:\s*=\s*\w+)?)*')

# Compile regex pattern for matching function declarations with arguments
# functionPattern = re.compile(r'function\s+(_\w+)\s*\(([_\w\s,]*)\)')
functionPattern = re.compile(r'(?:function\s+(_\w+)\s*\(([_\w\s,]*)\))|(?:function\s*\(([_\w\s,]*)\))')

# Compile regex pattern for matching for loop declarations with initial values
forLoopPattern = re.compile(r'for\s+(_\w+(?:,\s*_\w+)*)\s*')


def match_chunk_to_results(chunk_signatures, global_table_usage):
    """
    Matches chunk file function signatures to result file global function calls.
    chunk_signatures: List of tuples representing function signatures from the chunk file (n-th number, argument count).
    global_table_usage: Dict mapping global table calls from result files, including file name, n-th number, and arguments.
    """
    matched_results = {}


    for global_table, usages in list(global_table_usage.items()):
        usages = sorted(usages, key=lambda x: int(x[1]))
        counter = 0

        for usage in usages:
            result_file, result_nth, result_args = usage

            for chunk_name, chunk_data in chunk_signatures.items():
                missMatched = False
                # if number of function in chunk file is equal to number of function in result file
                # Edit: Actually according to chunk1582, this is not always the case
                for chunk_nth, chunk_args in chunk_data:
                    if str(chunk_nth) == result_nth:
                        chunk_args_count = len(chunk_args.split(',')) if chunk_args else 0
                        result_args_count = len(result_args.split(',')) if result_args else 0
                        if chunk_args_count == result_args_count:
                            counter += 1
                        else:
                            missMatched = True
                    else:
                        missMatched = True

                if (counter != 0 and len(usages) == counter):
                    # add chunk_name, global_table, usages to matched_results
                    matched_results[chunk_name] = (global_table, usages)

    return matched_results

# Example usage:
if __name__ == "__main__":
    functions_in_chunks = find_functions_in_files('Mappings')


    # Find Lua files in "results" folder
    results_lua_files = [os.path.join(root, file) for root, dirs, files in os.walk('./Decoded/') for file in files if file.endswith('.lua') if not "chunk" in file]

    # Now you can proceed to categorize by global table usage
    global_table_usage = categorize_by_global_table(results_lua_files)  # Limiting for demonstration

    # Define the path to the "updated" folder
    folder_path = "Mappings"

    updated_funcs = []
    # Iterate over all files in the "updated" folder
    for file_name in os.listdir(folder_path):
        if file_name.endswith(".lua"):  # Consider only Lua files
            file_content = read_file_content(folder_path, file_name)
            extracted_functions = extract_functions_from_lua(file_content)
            # print(f"Functions extracted from {file_name}:")
            print(extracted_functions)

            # if extracted_functions is [] then continue
            if not extracted_functions:
                continue

            chunk_functions = {file_name: functions_in_chunks[file_name],}

            # Call the function with your data
            matched_results = match_chunk_to_results(chunk_functions , global_table_usage)

            fun = matched_results[file_name]
            tableName, usages = fun
            for usage in usages:
                fileName, func_num, new_params = usage
                for index, params, body in extracted_functions:
                    if index == func_num:
                        # Replace the parameters in the function definition
                        # Find the index of the first occurrence of "(function"
                        index = body.find("(function")

                        # If "(function" exists, add "return" before it
                        if index != -1:
                            body = "return " + body[index:]
                        else:
                            print("No '(function' found in the script.")

                        # split new_params by comma and remove empty strings
                        new_params = list(filter(None, new_params.split(',')))

                        local_variable_replacements = {}
                        function_name_replacements = {}
                        function_argument_replacements = {}
                        loop_variable_replacements = {}

                        matchesLocal = localValPattern.finditer(body)
                        for matchLocal in matchesLocal:
                            original_names = matchLocal.groups()
                            for original_name in original_names:
                                if original_name is not None:
                                    if original_name not in local_variable_replacements:
                                        local_variable_replacements[original_name] = f"local_{len(local_variable_replacements) + 1}"
                                    body = body.replace(original_name, local_variable_replacements[original_name])

                        matchesFunction = functionPattern.finditer(body)
                        for matchFunction in matchesFunction:
                            function_name = matchFunction.group(1)
                            arguments_group1 = matchFunction.group(2)
                            arguments_group2 = matchFunction.group(3)

                            if arguments_group1:
                                arguments = arguments_group1.split(',')
                            elif arguments_group2:
                                arguments = arguments_group2.split(',')
                            else:
                                arguments = []

                            for argument in arguments:
                                argument = argument.strip()
                                if argument:
                                    if argument not in function_argument_replacements:
                                        function_argument_replacements[argument] = f"arg_{len(function_argument_replacements) + 1}"
                                    body = body.replace(argument, function_argument_replacements[argument])

                            if function_name:
                                if function_name not in function_name_replacements:
                                    function_name_replacements[function_name] = f"function_{len(function_name_replacements) + 1}"
                                body = body.replace(function_name, function_name_replacements[function_name])

                        matchesForLoop = forLoopPattern.finditer(body)
                        for matchForLoop in matchesForLoop:
                            loop_variables = matchForLoop.group(1).split(',')
                            for loop_variable in loop_variables:
                                loop_variable = loop_variable.strip()
                                if loop_variable not in loop_variable_replacements:
                                    loop_variable_replacements[loop_variable] = f"loop_{len(loop_variable_replacements) + 1}"
                                body = body.replace(loop_variable, loop_variable_replacements[loop_variable])


                        # Rename arguments
                        body = rename_arguments(body, new_params)
                        updated_funcs.append((fileName, body))

    # save files to a new folder of function_summaries
    if not os.path.exists('FinalResults'):
        os.makedirs('FinalResults')

    for fileName, body in updated_funcs:
        #print the file name and the body
        local_variable_replacements = {}
        function_name_replacements = {}
        function_argument_replacements = {}
        loop_variable_replacements = {}
        # Write the function body to a new file
        with open(os.path.join('FinalResults', fileName), 'w', encoding='utf-8') as file:
            file.write(body)

    # find the same key in matched_results and update the function parameters
    # Function to update function parameters based on the mapping
    def update_function_params(funcs, mappings):
        updated_funcs = []
        for fileName, variables in mappings.items():

            print(f"File: {fileName}")
            print(f"Variables: {variables}")
            tableName, usages = variables
            for usage in usages:
                fileName, func_num, new_params = usage

                for index, params, body in funcs:
                    if index == func_num:
                        # Find the index of the first occurrence of "(function"
                        index = body.find("(function")
                        # If "(function" exists, add "return" before it
                        if index != -1:
                            body = "return " + body[index:]
                        else:
                            print("No '(function' found in the script.")

                        # split new_params by comma and remove empty strings
                        new_params = list(filter(None, new_params.split(',')))

                        print(f"new_params: {new_params})")
                        # Rename arguments
                        body = rename_arguments(body, new_params)
                        # body = rename_locals(body)

                        print(body)
                        updated_funcs.append((fileName, body))
                        break

        return updated_funcs

```

This may not be the prettiest code that you have ever seen, but it works.

1. Replace all the encrypted keys with the actual keys.

2. Find the all unique encrypted table of functions.

3. Match with the actaul functions by using the number of arguments and unique keys.

4. Replace the arguments with the actual arguments.

5. Replace the local variables with the new local variables (ex. `local_1`, `local_2`, ...)

6. Save the results to the new folder.

7. Finally, you will get the deobfuscated scripts.

### Results

```lua
return (function(self)
    local local_1 = self["Get"](self)
    local local_2 = _UserService["LocalPlayer"]
    local local_3 = local_2["WsUser"]
    local_1["GetStatusBar"](local_1)["Hp"]["SetValue"](local_1["GetStatusBar"](local_1)["Hp"], local_3["Health"],
        local_3["MaxHealth"], _StatusBarElementTypes["Hp"])
    local local_4 = local_1["ControlWindowMan"]["Stat"]["StatWindowComponent"]
    local_4["Hp"]["UpdateHp"](local_4["Hp"])
end)
```

This is one of the deobfuscated scripts. As you can see, the encrypted keys are replaced with the actual keys.

Now, we can understand the script and analyze the script more easily without any confusion.

## Conclusion

In this post, I have explained how to deobfuscate the Lua script by using the mapping functions.

The mapping functions are the functions that map the keys to the actual keys.

By using the mapping functions, we can deobfuscate the Lua script and understand the script more easily.

I hope this post helps you to understand the Lua obfuscation and its deobfuscation.

If you have any questions or suggestions, please feel free to leave a comment below.

Thank you for reading!

## References

[^footnote]: Lua Official Website, [https://www.lua.org/about.html](https://www.lua.org/about.html)
[^fn-nth-2]: The 2nd footnote source
