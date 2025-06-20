## **Product Requirements Document: Block Builders**

Version: 1.0  
Date: June 19, 2025  
Author: Gemini Game Development  
Status: Draft

### **1\. Overview**

Product Name: Block Builders (working title)

Concept: A 3D sandbox, crafting, and survival game set in a world of floating islands. Players embody "Blocklys," creatures made of living blocks, who must build, explore, and form tribes to survive against environmental threats and mischievous creatures. The game blends creative construction with resource management, social dynamics, and light survival elements, designed for a broad audience including younger players (8+) and families.

Target Audience:

* Primary: Children and pre-teens (ages 8-14) who enjoy creative building games like Minecraft and Roblox.  
* Secondary: Families and casual gamers looking for a collaborative, non-hyper-violent creative outlet.

Core Pillars:

* Create: Build anything from simple shelters to complex machines on your personal floating island.  
* Explore: Discover new islands, resources, and creatures in an ever-expanding sky-world.  
* **Survive:** Protect your creations from mischievous Snelfs and the formidable Blockolf.  
* **Collaborate:** Connect with other players, trade resources, and form tribes for mutual benefit and survival.

### **2\. Game World & Lore**

**The Sky-World:** The game takes place in an endless sky, dotted with floating islands of various sizes, compositions, and biomes. These islands are made of "land blocks" (grass, dirt, stone) and are of unknown depth.

**Islands:**

* **Your Island:** Each player begins on a small, personal floating grassland island, given to them by their "parents" when they come of age. This is their home base.  
* **Drifting Islands:** Other islands (procedurally generated) drift through the sky. They can contain unique resources, biomes (e.g., desert islands, forest islands, crystal caves), and non-player characters (NPCs) or other players.  
* **Island Physics:** Islands have a "core" block. If the core is destroyed, the island's integrity might fail. Players can use special "Hook" items to temporarily connect their island to another, allowing for passage between them.

**Creatures & Characters:**

* **The Blockly (Player Character):** The player's avatar. It is visibly constructed from smaller blocks called "Liveblocks." A portion of a Blockly's Liveblocks can be given up in trades or to form bonds. The player's health or capabilities are tied to the number and quality of their Liveblocks.  
* **Companions:** Blocklys can use special "Core Liveblocks" found in the world to construct smaller, specialized companions (e.g., a "Digger Bot," a "Scout Bird") that can automate simple tasks.  
* **The Snelf:** A mischievous, neutral mob that is a mix between a sheep and an elf.  
  * **Behavior:** Snelfs are rare and hard to catch. They wander aimlessly and are generally passive but have a mischievous streak: they will randomly push loose blocks (not part of a structure) off the edge of the island.  
  * **Utility:** If a player manages to trap or tame a Snelf (e.g., by building a pen and offering it a rare type of flower), the Snelf will occasionally "sniff" the ground, providing a visual cue (e.g., a colored particle effect) that reveals the island's depth at that spot.  
* **The Blockolf (Primary Antagonist):** A formidable creature made of dark, corrupted blocks.  
  * **Behavior:** The Blockolf is the game's primary survival threat. It's not a constant presence but a recurring event. Its appearance is triggered by a timer (the "Blockolf Timer" in the MVP). When it appears, it actively seeks out and "eats" (destroys) blocks from the player's island.  
  * **Evolution:** The Blockolf has multiple forms. After eating a certain number of specific block types (e.g., 10 wood blocks, 5 stone blocks), it can evolve into a more powerful form with different abilities (e.g., a "Stone Blockolf" is harder to repel, a "Glow Blockolf" can move faster).  
  * **Repelling:** Players cannot kill the Blockolf initially, but they can repel it by hitting it, which "stuns" it and may cause it to de-spawn faster. Certain crafted items (e.g., a "Sound Spire" that emits a tone it dislikes) can act as deterrents.

### **3\. Core Gameplay Loop & Mechanics**

**Loop:** Build \> Survive \> Explore \> Collaborate \> Repeat

**Building & Crafting:**

* **Block System:** The world is grid-based. Blocks are smaller than in the MVP for more detailed creations.  
* **Inventory:** Players have a limited inventory for carrying blocks and items. Storage chests can be crafted.  
* **Resource Management:** Players start with a limited supply of blocks. They must acquire more by exploring other islands or carefully deconstructing parts of their own.  
* **Crafting Station:** A "Workbench" block is required to craft tools, items, and more complex blocks.  
* **Repairing Island Holes:** Digging completely through one's island creates a "sky-hole."  
  * **Repair Process:**  
    1. **Framework:** The player must craft and place a "Wooden Frame" block over the hole.  
    2. **Filling:** The frame must be filled with a "Compost Block" (crafted from dirt, seeds, and organic matter).  
    3. **Growth:** The compost block will slowly grow into a new "Grass Block" over time, but only if protected.  
    4. **Protection:** A "Wicker Fence" must be placed around the growing block. If any item or creature (especially a Snelf) falls onto the compost block, the process is reset.

**Survival Mechanics:**

* **The Blockolf Cycle:** The game operates on a day/night cycle, with the Blockolf's appearance being a major, telegraphed event (e.g., "The sky darkens... the Blockolf is near\!"). The initial objective is to build a specific structure (a shelter) before the first Blockolf arrives.  
* **Player Health (Liveblocks):** The player doesn't have a traditional health bar. Instead, taking damage from a fall or an enemy causes the Blockly to lose some of its Liveblocks, which fall to the ground. The player must pick them back up to "heal." If all core Liveblocks are lost, the player respawns.

**Exploration & Social Mechanics:**

* **Island Discovery:** Players can craft a "Spyglass" to see distant islands and a "Hook Launcher" to connect to them.  
* **Multiplayer & Tribes:**  
  * Players exist in a shared world space. They can visit each other's islands.  
  * **Negotiation:** Interaction with another Blockly is required before resources on their home island can be touched. Unauthorized block breaking is flagged as stealing and may have repercussions (e.g., NPC tribe guards become hostile).  
  * **Trading:** A dedicated UI allows players to securely trade blocks and items.  
  * **Forming a Tribe (Bonding):** To form a tribe, two Blocklys must perform a "Liveblock Swap." Each player gives a non-core Liveblock to the other. This forms a permanent bond, granting permissions to build on each other's islands and creating a shared tribe chat. It's a significant decision, as it permanently alters the composition of your own Blockly.

### **4\. Monetization (Conceptual)**

* **Model:** Buy-to-play (a single upfront purchase).  
* **Post-Launch:** Potentially offer purely cosmetic items (e.g., special skins/textures for your Blockly's Liveblocks, decorative items for your island) that do not affect gameplay balance. **No loot boxes or pay-to-win mechanics.**

### **5\. MVP (Minimum Viable Product) Feature Set**

* **Goal:** Validate the core loop of building a required structure under the threat of the Blockolf.  
* **Features:**  
  1. **World:** Single-player, non-persistent world that resets on game over. A flat "grassland" island.  
  2. **Player:** First-person controls for movement (Arrow Keys/WASD) and block placement/removal.  
  3. **Blocks:** A basic set of blocks (Grass, Stone, Wood).  
  4. **Inventory:** A simple, non-visual inventory system tracking the count of a limited starting supply of blocks. UI toolbar reflects counts.  
  5. **Objective:** A clearly defined structure to build (e.g., a 2x2 wood shelter).  
  6. **The Blockolf:**  
     * Replaces the simple timer.  
     * Appears visually as a simple placeholder (e.g., a large red cube) when its timer runs out.  
     * If the objective is not complete upon its arrival, it performs one "eat" action (destroys a key block) and the player loses.  
     * If the objective is complete, the player wins.  
  7. **UI:** Basic HUD displaying the objective, Blockolf timer, and block inventory in the toolbar. Win/Lose screens.  
  8. **API Integration:** "Get Hint" button using Gemini API to help players with the build objective.

### **6\. Future Development Roadmap (Post-MVP)**

* **Phase 1: Deepen Survival & World**  
  * Implement the full Blockolf behavior (multiple forms, repelling mechanics).  
  * Implement the "Snelf" creature and the island depth/repair mechanics.  
  * Introduce procedurally generated drifting islands with unique resources.  
  * Implement crafting (Workbench, Tools, Hooks).  
* **Phase 2: The Living World**  
  * Implement the visible "Blockly" player model and the Liveblock health system.  
  * Introduce crafting of "Companions."  
  * Introduce a basic day/night cycle.  
* **Phase 3: Multiplayer & Social**  
  * Enable multiplayer functionality (persistent shared world).  
  * Implement island visiting, permissions, and the "Hook Launcher."  
  * Implement the Trading UI and the "Liveblock Swap" mechanic for forming tribes.  
* **Phase 4: Expansion**  
  * New biomes, blocks, creatures, and crafting recipes.  
  * Tribe goals and collaborative build challenges.  
  * Story-driven quests and events.
