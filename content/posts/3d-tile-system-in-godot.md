---
title: "3D Tile System in Godot"
date: 2020-10-26T09:16:24+01:00
draft: false
description: "Simple 3D tile system in Godot"
tags: ["godot"]
cover:
    image: /blog/3d-tile-system-in-godot/cover.gif
ShowToc: false
TocOpen: false
---

This is a short article to bring some insight into how we implemented a simple 3D tile system in Godot. The code is in C#.

Enjoy!

## Overview

In essence, the entire map is just a 3D array where each cell maps to an abstract `Tile` class:

```csharp
Tile[,,] Tiles; // [column, row, floor]
```

The abstract `Tile` class defines the common data and behaviour that all tiles in the game should have. Arguably the most important piece of data that each tile has is its position in the 3D array, and its `Load` method which simply loads a 3D model into the game based on its cell position.

The tiles are constructed from a `JSON` file that defines the map. At runtime, this file is read and parsed into various tiles that inherits from the abstract `Tile` class and loaded into the game. That's basically it for the 3D tile system.

To demonstrate a small example, take this simple `JSON` file that defines two tiles in the game:

```json
{
    "columns": "2",
    "rows": "1",
    "floors": "1",
    "tiles": [
        {
            "type": "floor",
            "cell": { "column": 0, "row": 0, "floor": 0 },
            "orientation": "south",
            "model": {
                "roof": "res://assets/tile/grass-flat.glb",
                "block": "res://assets/tile/dirt-block.glb"
            },
            "height": 2,
            "height_offset": 0
        },
        {
            "type": "ramp",
            "cell": { "column": 1, "row": 0, "floor": 1 },
            "orientation": "east",
            "model": {
                "roof": "res://assets/tile/grass-slope.glb",
                "block": "res://assets/tile/dirt-slope.glb"
            },
            "height": 2,
            "height_offset": 0
        }
    ]
}
```

First we define how big the map is by the entries at the top. From there, we simply define a list of tiles that we want specified by its `type`. The `floor` type will create an instance of a `FloorTile` class that allows a player to walk in any direction (i.e. north, east, south or west). The `ramp` type spawns a `RampTile` instance that allows a player to go either up or down a floor, and can only be accessed from two directions; either north and south, or west and east.

The `JSON` file produces the following tiny map:

![](/blog/3d-tile-system-in-godot/tiny-map.png)


## The Board Class

The `Board` class is sort of what connects everything together. It generates the map and allows the players and tiles to interact with both the board and each other. Let's take a look at some of its functionality:

```csharp
public class Board {
	public static readonly int CELL_SIZE = 2;

	public Tile[,,] Tiles { get; private set; }
	public int Columns { get; private set; }
	public int Rows { get; private set; }
	public int Floors { get; private set; }

    public Board(string jsonFile) {
		var json = Read(jsonFile);
		Tiles = Parse(json);

		Columns = Tiles.GetLength(0);
		Rows = Tiles.GetLength(1);
		Floors = Tiles.GetLength(2);
    }

    public Tile GetTile(BoardCell cell) {
		return Tiles[cell.Column, cell.Row, cell.Floor];
	}

    public void Load(Node parent) {
		for (int floor = 0; floor < Floors; floor++) {
			for (int row = 0; row < Rows; row++) {
				for (int col = 0; col < Columns; col++) {
					var tile = Tiles[col, row, floor];
					tile?.Load(parent);
				}
			}
		}
	}

    private string Read(string jsonFile) {
		// Read JSON file
	}

	private Tile[,,] Parse(string json) {
		// Parse JSON file
	}
```

The `Board` class is instanced from the root node in a scene, which then calls the `Load` method and passes itself as `parent` so that tiles can add a child to it. The `BoardCell` type refers to a simple utility struct:

```csharp
public readonly struct BoardCell {
	public readonly int Column, Row, Floor;
	
	public BoardCell(int column, int row, int floor) {
		this.Column = column;
		this.Row = row;
		this.Floor = floor;
	}

	public Vector3 ToWorld() {
		return new Vector3(Column, Floor, Row) * Board.CELL_SIZE;
	}

    /* ... */
}
```

Every loaded tile in the game must be set to a proper position in the scene's world space. This is where the `ToWorld()` comes in. To make life easy, two key assumptions are made:

* Every tile has the same size.
* The [object origin](https://docs.blender.org/manual/en/latest/scene_layout/object/origin.html) of every 3D model is located at the bottom center.

This makes the mapping from a cell position to world position quite easy when it comes to positioning a tile. That brings us to the abstract `Tile` class, which has one key piece of data and one key method:

```csharp
public abstract class Tile {
    public BoardCell Cell;
    /* ... */
    public virtual void Load(Node parent) {
		// Load tile to scene tree.
        // This basically means to load and instance a 3D model,
        // and add it as a child to parent.
        // Then set its position with Cell.ToWorld().
	}
}
```

## Moving A Player

To move a player, we need some sort of pathfinding or _graph search_ algorithm which you can read more about from [this](https://www.redblobgames.com/pathfinding/a-star/introduction.html) wonderful article. To solve this, we introduce the `TileWalkable` class that provides two key methods:

```csharp
public abstract class TileWalkable : Tile {
    public abstract List<TileWalkable> Neighbors(Tile[,,] grid);

    public bool IsConnected(TileWalkable tile, Tile[,,] grid) {
		return this.Neighbors(grid).Contains(tile);
	}
}
```

The `IsConnected` converts the 3D grid array to a graph where various tiles are connected, and the `Neighbors` method provides the logic for which neighbors a tile actually has. Together these two methods will allow us to make use of a graph search algorithm.

Let's take a look at the `FloorTile` class to shed some light on this:

```csharp
public class FloorTile : TileWalkable {
	public override List<TileWalkable> Neighbors(Tile[,,] grid) {
		var neighbors = new List<TileWalkable>();

		var dirs = new List<TileOrientation>() {
			TileOrientation.North,
			TileOrientation.East,
			TileOrientation.South,
			TileOrientation.West
		};

		foreach (var dir in dirs) {
			var adjacent = Cell.Adjacent(dir);
			if (adjacent.IsOutsideGrid(grid)) continue;

			var tile = grid[adjacent.Column, adjacent.Row, adjacent.Floor];
			if (tile is TileWalkable walkable) {
				neighbors.Add(walkable);
			}
		}

		return neighbors;
	}
}
```

Recall that a `FloorTile` allows a player to walk in any direction, and so the goal is to find the adjacent tile for each direction and check if that is also a tile that a player can walk on. If it is, we add it to the `neighbors` list. The `TileOrientation` is just an `enum`, and the `Cell.Adjacent(dir)` method is a nice utility function from the `BoardCell` struct to make the code more readable. The code for it is as follows:

```csharp
public BoardCell Adjacent(TileOrientation orientation, int floorOffset = 0) {
    int column = Column;
    int row = Row;
    int floor = Floor + floorOffset;

    switch (orientation) {
        case TileOrientation.North: row -= 1; break;
        case TileOrientation.South: row += 1; break;
        case TileOrientation.West: column -= 1; break;
        case TileOrientation.East: column += 1; break;
    }

    return new BoardCell(column, row, floor);
}
```

With the `IsConnected` and `Neighbors` methods in place, we can then make use of them with a graph search algorithm in the `Board` class. Following any of the algorithms from the mentioned [article](https://www.redblobgames.com/pathfinding/a-star/introduction.html) earlier, we get the following:

```csharp
public List<TileWalkable> ShortestPath(TileWalkable from, TileWalkable to) {
    /* ... */
    while (frontier.Count > 0) {
        var current = frontier.Dequeue();
        /* ... */
        foreach (var neighbor in current.Neighbors(Tiles)) {
            if (!neighbor.IsConnected(current, board.Tiles)) continue;
            /* ... */
        }
    }

    return new List<TileWalkable>();
}
```

This will return a list of `TileWalkable` tiles which represents the shortest path from one tile to another. To finally make a player move, one only needs to construct a list of `Vector3` positions from the tiles and make the player follow that path.