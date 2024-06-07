+++
title = 'How to parse an array with differing array types'
tags = ["golang"]
+++

# How to parse an array which contains dynamic types

Parsing an array which has a single type within it is easy, but what about arrays which have dynamic values? Parsing such arrays could be tricky but in this post we will explore how to do just that. Lets imagine we are making a game and we have an array or nearby targets, each one having different attributes based on the type of nearby target.

## Our array
```json
{
    "targets": [
        {"type": "player", "attributes": {"level": 10, "name": "Duke", "class": "Warrior"}},
        {"type": "enemy", "attributes": {"level": 11, "name": "Goblin", "personality": "aggressive"}},
        {"type": "object", "attributes": {"name": "Tree", interactable": false }}
    ]
}
```
In the JSON above we have an array which has 3 items in it. Each item represents a different kind of targets which our player can do things with.

## Parsing the array

### Types

Lets first create our classes
```golang
type Player struct {
    Level int `json:"level"`
    Name string `json:"name"`
    Class string `json:"class"`
}

type Enemy struct {
    Level int `json:"level"`
    Name string `json:"name"`
    Personality string `json:"personality"`
}

type Object struct {
    Name string `json:"name"`
    Interactable bool `json:"interactable"`
}
```

Each of our classes now can be used to parse their own individual JSON string, but when they are all in an array it won't really work. Lets make our lives a bit easier and make an interface which all of our types will implement.

```golang
type Named interface {
    GetName() string
}

func (p Player) GetName() string {
    return p.Name
}

func (e Enemy) GetName() string {
    return g.Name
}

func (o Object) GetName() string {
    return o.Name
}
```

## Lets start parsing

Now that we have an interface which all of our types implement. We can create an array which uses that interface. But still how do we parse the values? The way we can parse the json is a combination of anonymous structs and `json.RawMessage`. Lets start Unmarshaling

```golang
func main() {
    j := []byte(`{
        "targets": [
            {"type": "player", "attributes": {"level": 10, "name": "Duke", "class": "Warrior"}},
            {"type": "enemy", "attributes": {"level": 11, "name": "Goblin", "personality": "aggressive"}},
            {"type": "object", "attributes": {"name": "Tree", "interactable": false }}
        ]
    }`)

    var nameableTargets struct{
        Targets []Named // Our end goal is to fill this array
    }

    var targets struct{
        Targets []struct{
            // Depending on the value of this field we will be determining what kind of struct we should initiate
            Type string `json:"type"`
            
            // By using RawMessage we can delay the parsing of the values in order to determine what kind of type to use
            Arguments json.RawMessage `json:"attributes"` 
        }
    }
    
    // Parse the json into our temporary collection
    if err := json.Unmarshal(j, &targets); err != nil {
        panic(err)
    }
    
    // Fill out the result array with the correct types
    for _, t := range targets.Targets {
        switch t.Type {
        case "player":
            var player Player
            if err := json.Unmarshal(t, &player); err != nil {
                panic(err)
            }
            nameableTargets = append(nameableTargets, player)
        case "enemy":
            var enemy Enemy
            if err := json.Unmarshal(t, &enemy); err != nil {
                panic(err)
            }
            nameableTargets = append(nameableTargets, enemy)
        case "object":
            var obj Object
            if err := json.Unmarshal(t, &object); err != nil {
                panic(err)
            }
            nameableTargets = append(nameableTargets, obj)
        }
    }
    
    fmt.Printf("%+v", nameableTargets)
}
```

And that is the way we can dynamically determine what kind of object should be created and saved to the array.