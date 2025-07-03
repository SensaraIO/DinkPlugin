# Dink Webhook Integration for Next.js Servers

This document outlines how to receive and process webhook events from Dink on a Next.js server. Dink webhooks provide real-time notifications for various in-game events in Old School RuneScape.

## 1. Basic Next.js API Route Setup

To receive Dink webhooks, you'll need to create an API route in your Next.js application.

**a. Create the API Route File:**

Create a file in your `pages/api` directory. For example, `pages/api/dink-webhook.js` (or `.ts` if using TypeScript).

```javascript
// pages/api/dink-webhook.js

export default async function handler(req, res) {
  if (req.method === 'POST') {
    // Process the webhook
    // ...
    res.status(200).json({ message: 'Webhook received' });
  } else {
    // Handle any other HTTP methods
    res.setHeader('Allow', ['POST']);
    res.status(405).end(`Method ${req.method} Not Allowed`);
  }
}
```

**b. Parsing `multipart/form-data`:**

Dink sends webhooks with a `Content-Type` of `multipart/form-data`. This is because payloads can include both a JSON object (`payload_json`) and an optional image file (`file` for screenshots).

You'll need a library to parse this data format in your API route. Standard Next.js API routes (Node.js runtime) do not parse `multipart/form-data` by default.

*   **For Node.js Runtime (Default):**
    *   [`formidable`](https://www.npmjs.com/package/formidable): A popular choice for parsing form data, including files.
    *   [`busboy`](https://www.npmjs.com/package/busboy): A streaming parser, can be more efficient for large files. You'd typically use it with a wrapper like `connect-busboy`.
    *   [`multer`](https://www.npmjs.com/package/multer): While often used with Express, its core can be adapted or you can find Next.js-specific wrappers. However, `formidable` is often more straightforward for simple API routes.

    **Example with `formidable`:**
    First, install it: `npm install formidable` or `yarn add formidable`.

    Then, you'll need to disable Next.js's default body parser for this route and use `formidable`:

    ```javascript
    // pages/api/dink-webhook.js
    import formidable from 'formidable';

    // Disable Next.js's default body parser for this route
    export const config = {
      api: {
        bodyParser: false,
      },
    };

    export default async function handler(req, res) {
      if (req.method === 'POST') {
        const form = formidable({}); // or new formidable.IncomingForm(); in older versions

        try {
          const [fields, files] = await form.parse(req);

          const payloadJsonString = fields.payload_json?.[0]; // formidable wraps fields in arrays
          const imageFile = files.file?.[0]; // formidable wraps files in arrays

          if (payloadJsonString) {
            const webhookData = JSON.parse(payloadJsonString);
            // Now process webhookData and potentially imageFile
            console.log('Received Dink Webhook Event:', webhookData.type);
            // Your custom logic here

            res.status(200).json({ message: 'Webhook received successfully' });
          } else {
            res.status(400).json({ error: 'Missing payload_json' });
          }
        } catch (error) {
          console.error('Error parsing form data:', error);
          res.status(500).json({ error: 'Error processing webhook' });
        }
      } else {
        res.setHeader('Allow', ['POST']);
        res.status(405).end(`Method ${req.method} Not Allowed`);
      }
    }
    ```

*   **For Edge Runtime:**
    If you configure your API route to use the Edge runtime, you can use the standard `request.formData()` method:

    ```javascript
    // pages/api/dink-webhook.js (with Edge runtime)
    export const config = {
      runtime: 'edge', // or 'experimental-edge'
    };

    export default async function handler(req) {
      if (req.method === 'POST') {
        try {
          const formData = await req.formData();
          const payloadJsonString = formData.get('payload_json');
          const imageFile = formData.get('file'); // This will be a File object

          if (payloadJsonString) {
            const webhookData = JSON.parse(payloadJsonString);
            // Now process webhookData and potentially imageFile
            console.log('Received Dink Webhook Event:', webhookData.type);
            // Your custom logic here

            return new Response(JSON.stringify({ message: 'Webhook received successfully' }), {
              status: 200,
              headers: { 'Content-Type': 'application/json' },
            });
          } else {
            return new Response(JSON.stringify({ error: 'Missing payload_json' }), {
              status: 400,
              headers: { 'Content-Type': 'application/json' },
            });
          }
        } catch (error) {
          console.error('Error parsing form data:', error);
          return new Response(JSON.stringify({ error: 'Error processing webhook' }), {
            status: 500,
            headers: { 'Content-Type': 'application/json' },
          });
        }
      } else {
        return new Response(`Method ${req.method} Not Allowed`, {
          status: 405,
          headers: { Allow: 'POST' },
        });
      }
    }
    ```
    Choose the runtime and parsing method that best suits your project needs. The Node.js runtime with `formidable` is often more common for existing Next.js projects.

---

## 2. General Webhook Processing Logic

Once you have parsed the `multipart/form-data`, you can access the webhook's content.

**a. Accessing `payload_json`:**
The core data of the webhook is a JSON string within the `payload_json` form field.

   *   If using `formidable` (as in the Node.js example above), `fields.payload_json[0]` will give you this string.
   *   If using `req.formData()` (Edge runtime), `formData.get('payload_json')` will give you this string.

**b. Parsing `payload_json`:**
This string needs to be parsed into a JavaScript object:

   ```javascript
   const webhookData = JSON.parse(payloadJsonString);
   ```

**c. Accessing the Optional Screenshot (`file`):**
If a screenshot is included with the webhook, it will be in the `file` form field.

   *   With `formidable`, `files.file[0]` will be an object representing the uploaded file (includes properties like `filepath`, `originalFilename`, `mimetype`, etc.). You can then read the file from its temporary `filepath`.
   *   With `req.formData()` (Edge runtime), `formData.get('file')` will be a `File` object. You can access its contents using methods like `imageFile.arrayBuffer()`.

   **Note:** Always check if `imageFile` exists before trying to process it, as screenshots are optional.

**d. Identifying the Event Type:**
The most important field in the parsed `webhookData` is `type`. This string tells you what kind of event occurred (e.g., "LOOT", "DEATH", "LEVEL").

   ```javascript
   const eventType = webhookData.type;
   console.log("Event Type:", eventType);
   ```
   You will use `eventType` to decide how to handle the specific data for that event.

**e. Using the `extra` Field for Data:**
While the `webhookData` object contains fields like `content` (user-defined message) and `embeds` (for Discord), **it is highly recommended to use the `extra` object for reliable and structured data extraction.**

   ```javascript
   const eventDetails = webhookData.extra;
   // eventDetails will contain the specific data for the eventType
   // e.g., if eventType is "LOOT", eventDetails will have "items", "source", etc.
   ```
   The structure of the `extra` object varies depending on the `eventType`. The following sections will detail these structures.

**f. Base Data:**
All webhooks will also include these top-level fields in `webhookData` (outside `extra`), which can be useful:
    *   `playerName`: The player's RuneScape name.
    *   `accountType`: e.g., "NORMAL", "IRONMAN".
    *   `seasonalWorld`: "true" or "false" (string).
    *   `dinkAccountHash`: A persistent unique identifier for the player, useful for tracking users across name changes.
    *   And potentially optional fields like `clanName`, `groupIronClanName`, `discordUser`, `world`, `regionId` if enabled in Dink settings.

---

## 3. Webhook Event Types and `extra` Payload Structures

This section details the different webhook `type` values and the structure of their corresponding `extra` object. Always refer to the `type` field in the main webhook body to determine which structure to expect in the `extra` field.

The common top-level fields (`playerName`, `accountType`, `dinkAccountHash`, etc.) are available for all types but are not repeated in each example below.

---

### `type: "DEATH"`
Signifies that the player has died in the game.

**`extra` Object Example (NPC Death):**
```json
{
  "valueLost": 300,
  "isPvp": false,
  "killerName": "Goblin", // Can be NPC name or player name if isPvp is true
  "killerNpcId": 69, // Only if killed by NPC
  "keptItems": [], // Array of item objects
  "lostItems": [    // Array of item objects
    {
      "id": 314,
      "quantity": 100,
      "priceEach": 3,
      "name": "Feather"
    }
  ],
  "location": { // Optional, if location is enabled
    "regionId": 10546,
    "plane": 0,
    "instanced": false
  }
}
```
**Key `extra` Fields:**
*   `valueLost`: Total estimated value of items lost.
*   `isPvp`: Boolean, true if killed by another player.
*   `killerName`: Name of the PKer (if `isPvp` is true) or NPC.
*   `killerNpcId`: NPC ID of the killer, if applicable.
*   `keptItems`: Array of items the player kept. Each item is an object with `id`, `quantity`, `priceEach`, `name`.
*   `lostItems`: Array of items the player lost. Same structure as `keptItems`.
*   `location`: Object with `regionId`, `plane`, `instanced` boolean (if location sharing is enabled).

---

### `type: "COLLECTION"`
Signifies an item has been added to the player's collection log.

**`extra` Object Example:**
```json
{
  "itemName": "Zamorak chaps",
  "itemId": 10372,
  "price": 500812,
  "completedEntries": 420,
  "totalEntries": 1443,
  "currentRank": "IRON",
  "rankProgress": 120,
  "logsNeededForNextRank": 80,
  "nextRank": "STEEL",
  "justCompletedRank": "BRONZE", // Optional
  "dropperName": "Clue Scroll (Hard)", // Optional
  "dropperType": "EVENT", // Optional, e.g., EVENT, NPC
  "dropperKillCount": 1500 // Optional
}
```
**Key `extra` Fields:**
*   `itemName`, `itemId`, `price`: Details of the collected item.
*   `completedEntries`, `totalEntries`: Player's collection log progress.
*   `currentRank`, `rankProgress`, `logsNeededForNextRank`, `nextRank`: Info about collection log tiers/ranks.
*   `justCompletedRank`: Present if a rank was just completed.
*   `dropperName`, `dropperType`, `dropperKillCount`: Information about the source of the item, if available.

---

### `type: "LEVEL"`
Signifies a skill has leveled up.

**`extra` Object Example:**
```json
{
  "levelledSkills": {
    "Attack": 76,
    "Strength": 80
  },
  "allSkills": {
    "Attack": 76, "Strength": 80, "Defence": 70, ... // All skills and their levels
  },
  "combatLevel": {
    "value": 95,
    "increased": true // boolean, true if combat level changed
  }
}
```
**Key `extra` Fields:**
*   `levelledSkills`: An object where keys are skill names and values are their new levels (for skills that just leveled up).
*   `allSkills`: An object containing all skills and their current levels.
*   `combatLevel`: An object with `value` (current combat level) and `increased` (boolean).

---

### `type: "XP_MILESTONE"`
Signifies a skill has reached a user-defined XP milestone.

**`extra` Object Example:**
```json
{
  "xpData": {
    "Agility": 5000000,
    "Herblore": 10000000
  },
  "milestoneAchieved": ["Agility", "Herblore"], // Skill(s) that hit a milestone
  "interval": 5000000 // The configured XP interval that was met
}
```
**Key `extra` Fields:**
*   `xpData`: Object mapping skill names to their total XP.
*   `milestoneAchieved`: Array of skill names that hit the XP milestone.
*   `interval`: The XP interval value that was triggered.

---

### `type: "LOOT"`
Signifies valuable loot received from an activity (NPC kill, skilling, clue, etc.).

**`extra` Object Example:**
```json
{
  "items": [
    {
      "id": 1234,
      "quantity": 1,
      "priceEach": 42069,
      "name": "Some item",
      "criteria": ["VALUE"], // Why this item triggered notification (e.g., VALUE, RARITY, UNIQUE)
      "rarity": 0.005 // Optional, decimal probability if available
    }
  ],
  "source": "Tombs of Amascut", // Name of the loot source
  "party": ["Player1", "Player2"], // Optional, array of player names in a party (e.g., raids)
  "category": "EVENT", // Type of loot source, e.g., NPC, EVENT, PICKPOCKET. See LootRecordType enum.
  "killCount": 60, // Optional, KC for the source if available
  "rarestProbability": 0.001, // Optional, probability of the rarest item in the loot pile
  "npcId": null // Optional, ID of the NPC if category is NPC or PICKPOCKET
}
```
**Key `extra` Fields:**
*   `items`: Array of item objects. Each item has `id`, `quantity`, `priceEach`, `name`, `criteria` (array of reasons it's notable), and optional `rarity`.
*   `source`: Name of the activity or NPC that yielded the loot.
*   `party`: Array of player names involved, if applicable (e.g., for raids).
*   `category`: String indicating the type of loot record (e.g., "NPC", "EVENT", "PICKPOCKET").
*   `killCount`: Kill count for the `source`, if available.
*   `rarestProbability`: Rarity of the most rare item in the drop, if applicable.
*   `npcId`: ID of the NPC, if applicable.

---

### `type: "SLAYER"`
Signifies completion of a slayer task.

**`extra` Object Example:**
```json
{
  "slayerTask": "Kill 120 Blue Dragons",
  "slayerCompleted": "250", // Total tasks completed (string)
  "slayerPoints": "25",   // Points awarded (string)
  "killCount": 120,       // Number of creatures killed for this task
  "monster": "Blue Dragon"  // Name of the monster
}
```
**Key `extra` Fields:**
*   `slayerTask`: Full name of the completed task.
*   `slayerCompleted`: Total number of slayer tasks completed by the player (as a string).
*   `slayerPoints`: Slayer points awarded for this task (as a string).
*   `killCount`: The number of monsters assigned for the task.
*   `monster`: The name of the slayer monster.

---

### `type: "QUEST"`
Signifies completion of a quest.

**`extra` Object Example:**
```json
{
  "questName": "Dragon Slayer I",
  "completedQuests": 22,
  "totalQuests": 156,
  "questPoints": 44,
  "totalQuestPoints": 293
}
```
**Key `extra` Fields:**
*   `questName`: Name of the completed quest.
*   `completedQuests`: Number of quests the player has now completed.
*   `totalQuests`: Total quests available in the game.
*   `questPoints`: Quest points the player now has.
*   `totalQuestPoints`: Total quest points available in the game.

---

### `type: "CLUE"`
Signifies completion of a clue scroll.

**`extra` Object Example:**
```json
{
  "clueType": "Hard", // e.g., Beginner, Easy, Medium, Hard, Elite, Master
  "numberCompleted": 123,
  "items": [
    {
      "id": 1234,
      "quantity": 1,
      "priceEach": 42069,
      "name": "Some item"
    }
    // ... more items
  ]
}
```
**Key `extra` Fields:**
*   `clueType`: The tier of the completed clue scroll.
*   `numberCompleted`: How many clues of this tier the player has completed.
*   `items`: Array of item objects received as reward. Each item has `id`, `quantity`, `priceEach`, `name`.

---

### `type: "KILL_COUNT"`
Signifies a new kill count for a boss or activity, potentially a personal best (PB).

**`extra` Object Example:**
```json
{
  "boss": "Chambers of Xeric",
  "count": 69,
  "gameMessage": "Your completed Chambers of Xeric count is: 69.",
  "time": "PT46M34S", // Optional, ISO-8601 duration format for completion time
  "isPersonalBest": true, // Optional, boolean
  "personalBest": null, // Optional, ISO-8601 duration of previous PB if current is not PB
  "party": ["Player1", "Player2"] // Optional, array of player names
}
```
**Key `extra` Fields:**
*   `boss`: Name of the boss or activity.
*   `count`: The new kill/completion count.
*   `gameMessage`: The in-game message announcing the KC.
*   `time`: Completion time for this kill, if applicable (ISO-8601 duration string, e.g., "PT1M30S").
*   `isPersonalBest`: True if this completion was a personal best time.
*   `personalBest`: Previous personal best time, if `time` is present but `isPersonalBest` is false.
*   `party`: List of players in the party, if applicable (e.g. for raids).

---

### `type: "COMBAT_ACHIEVEMENT"`
Signifies completion of a combat achievement task or tier.

**`extra` Object Example (Task Completion):**
```json
{
  "tier": "GRANDMASTER", // Tier of the completed task
  "task": "Peach Conjurer", // Name of the task
  "taskPoints": 6,
  "totalPoints": 1337, // Player's total CA points
  "tierProgress": 517, // Points within the current tier
  "tierTotalPoints": 645, // Total points for the current tier
  "totalPossiblePoints": 14814, // Max CA points in game
  "currentTier": "MASTER", // Tier player is currently in
  "nextTier": "GRANDMASTER", // Next tier
  "justCompletedTier": "MASTER" // Optional, if this task completed a tier
}
```
**Key `extra` Fields:**
*   `tier`: Difficulty tier of the completed task (e.g., "EASY", "GRANDMASTER").
*   `task`: Name of the specific combat task completed.
*   `taskPoints`: Points awarded for this specific task.
*   `totalPoints`: Player's new total combat achievement points.
*   `tierProgress`, `tierTotalPoints`: Progress towards completing the `currentTier`.
*   `totalPossiblePoints`: Total CA points available in game.
*   `currentTier`, `nextTier`: Current and next CA tier for the player.
*   `justCompletedTier`: If completing this task also completed a tier, this field will indicate which tier was completed (e.g., "HARD").

---

### `type: "ACHIEVEMENT_DIARY"`
Signifies completion of an achievement diary task or an entire diary tier.

**`extra` Object Example:**
```json
{
  "area": "Varrock", // Geographic area of the diary
  "difficulty": "HARD", // Difficulty of the diary
  "total": 15, // Total number of diary tiers completed by player
  "tasksCompleted": 152, // Total tasks completed by player across all diaries
  "tasksTotal": 492, // Total tasks possible in game
  "areaTasksCompleted": 37, // Tasks completed by player in this specific area/difficulty
  "areaTasksTotal": 42 // Total tasks in this specific area/difficulty
}
```
**Key `extra` Fields:**
*   `area`: The geographical area of the diary (e.g., "Lumbridge", "Varrock").
*   `difficulty`: The difficulty tier of the diary (e.g., "EASY", "MEDIUM", "HARD", "ELITE").
*   `total`: Total number of diary *tiers* (Area+Difficulty combinations) completed by the player.
*   `tasksCompleted`: Total number of individual diary *tasks* completed by the player across all diaries.
*   `tasksTotal`: Total individual diary tasks available in the game.
*   `areaTasksCompleted`: Number of tasks completed for the specific `area` and `difficulty`.
*   `areaTasksTotal`: Total tasks possible for the specific `area` and `difficulty`.

---

### `type: "PET"`
Signifies that the player has received a pet.

**`extra` Object Example:**
```json
{
  "petName": "Ikkle hydra", // Name of the pet
  "milestone": "5,000 killcount", // Optional, if pet was from a KC milestone clan message
  "duplicate": false // Optional, boolean, true if it's a duplicate pet
}
```
**Key `extra` Fields:**
*   `petName`: The name of the pet received.
*   `milestone`: If the pet drop coincided with a kill count milestone announced in clan chat (e.g., "1000th kill").
*   `duplicate`: Boolean indicating if this is a duplicate pet for the player.

---

### `type: "SPEEDRUN"`
Signifies completion of a quest speedrun.

**`extra` Object Example (Personal Best):**
```json
{
  "questName": "Cook's Assistant",
  "personalBest": "1:13.20", // Current PB (which is this run)
  "currentTime": "1:13.20", // Time for this run
  "isPersonalBest": true
}
```
**Key `extra` Fields:**
*   `questName`: Name of the quest for the speedrun.
*   `personalBest`: The player's personal best time for this quest (string format like "MM:SS.ss").
*   `currentTime`: The time achieved in this specific run.
*   `isPersonalBest`: Boolean, true if this run was a new personal best.

---

### `type: "BARBARIAN_ASSAULT_GAMBLE"`
Signifies a gamble at Barbarian Assault, often for a Penance pet or item.

**`extra` Object Example:**
```json
{
  "gambleCount": 500,
  "items": [ // Items received from the gamble, if any notable ones
    {
      "id": 3122,
      "quantity": 1,
      "priceEach": 35500,
      "name": "Granite shield"
    }
  ]
}
```
**Key `extra` Fields:**
*   `gambleCount`: The player's total high-level gamble count at Barbarian Assault.
*   `items`: An array of item objects received from the gamble, if any items met notification criteria. Structure is `id`, `quantity`, `priceEach`, `name`.

---

### `type: "PLAYER_KILL"`
Signifies that the logged-in player has killed another player in PvP.

**`extra` Object Example:**
```json
{
  "victimName": "PvP Target",
  "victimCombatLevel": 69,
  "victimEquipment": { // Object mapping equipment slot (string) to item object
    "AMULET": {"id": 1731, "priceEach": 1987, "name": "Amulet of power"},
    "WEAPON": {"id": 1333, "priceEach": 14971, "name": "Rune scimitar"}
    // ... other slots like TORSO, LEGS, HANDS, etc.
  },
  "world": 394, // Optional
  "location": { // Optional
    "x": 3334,
    "y": 4761,
    "plane": 0
  },
  "myHitpoints": 20, // Player's HP after the kill
  "myLastDamage": 12 // Damage of the killing blow
}
```
**Key `extra` Fields:**
*   `victimName`: Name of the player killed.
*   `victimCombatLevel`: Combat level of the victim.
*   `victimEquipment`: An object where keys are equipment slot names (e.g., "AMULET", "WEAPON", "CAPE", "RING", "BOOTS", "SHIELD", "HEAD", "TORSO", "LEGS", "HANDS") and values are item objects (`id`, `priceEach`, `name`).
*   `world`, `location`: Optional, if location sharing is enabled.
*   `myHitpoints`: The attacker's (your player's) hitpoints after the kill.
*   `myLastDamage`: The damage dealt by the attacker's killing blow.


---

### `type: "GROUP_STORAGE"`
Signifies items being deposited to or withdrawn from a Group Ironman shared bank.

**`extra` Object Example:**
```json
{
  "groupName": "Iron Buddies",
  "deposits": [
    {"id": 315, "name": "Shrimps", "quantity": 2, "priceEach": 56}
  ],
  "withdrawals": [
    {"id": 1265, "name": "Bronze pickaxe", "quantity": 1, "priceEach": 22}
  ],
  "netValue": 143 // Value of deposits minus value of withdrawals for this event
}
```
**Key `extra` Fields:**
*   `groupName`: Name of the Group Ironman group.
*   `deposits`: Array of item objects deposited. Each item: `id`, `name`, `quantity`, `priceEach`.
*   `withdrawals`: Array of item objects withdrawn. Same structure.
*   `netValue`: The net change in value in the group storage from this transaction.

---

### `type: "GRAND_EXCHANGE"`
Signifies an update to a Grand Exchange offer (e.g., bought, sold, cancelled).

**`extra` Object Example:**
```json
{
  "slot": 1, // 1-indexed GE slot number
  "status": "SOLD", // e.g., BOUGHT, SOLD, CANCELLED, IN_PROGRESS
  "item": {
    "id": 314,
    "quantity": 2, // Quantity transacted in this update
    "priceEach": 3, // Price per item in this transaction
    "name": "Feather"
  },
  "marketPrice": 2, // Current market price of the item
  "targetPrice": 3, // The price the offer was set at
  "targetQuantity": 2, // Total quantity of the offer
  "sellerTax": 0 // Tax paid if item was sold
}
```
**Key `extra` Fields:**
*   `slot`: The GE slot number (1-indexed).
*   `status`: The state of the offer (e.g., "BOUGHT", "SOLD", "CANCELLED"). See RuneLite's `GrandExchangeOfferState` for all possible values.
*   `item`: An object describing the item involved: `id`, `quantity` (transacted in this update), `priceEach` (actual transaction price), `name`.
*   `marketPrice`: The guide price of the item at the time of the event.
*   `targetPrice`: The price the offer was originally listed for.
*   `targetQuantity`: The total quantity of the offer as originally listed.
*   `sellerTax`: The amount of tax paid if the status is "SOLD".

---

### `type: "TRADE"`
Signifies a completed trade with another player.

**`extra` Object Example:**
```json
{
  "counterparty": "TradingPartner", // Name of the other player in the trade
  "receivedItems": [
    {"id": 314, "quantity": 100, "priceEach": 2, "name": "Feather"}
  ],
  "givenItems": [
    {"id": 2, "quantity": 3, "priceEach": 150, "name": "Cannonball"}
  ],
  "receivedValue": 200, // Total value of items received by your player
  "givenValue": 450     // Total value of items given by your player
}
```
**Key `extra` Fields:**
*   `counterparty`: The name of the other player involved in the trade.
*   `receivedItems`: Array of item objects your player received. Each item: `id`, `quantity`, `priceEach`, `name`.
*   `givenItems`: Array of item objects your player gave. Same structure.
*   `receivedValue`: Total market value of items your player received.
*   `givenValue`: Total market value of items your player gave.

---

### Leagues Event Types
These types are specific to Leagues gameplay. The `accountType` will typically be "IRONMAN" and `seasonalWorld` will be "true".

**`type: "LEAGUES_AREA"`** (Region Unlocked)
```json
{
  "area": "Kandarin",
  "index": 2, // Order of unlock (0 = Karamja, 1 = first choice, etc.)
  "tasksCompleted": 200,
  "tasksUntilNextArea": 200 // Optional
}
```

**`type: "LEAGUES_MASTERY"`** (Combat Mastery Unlocked)
```json
{
  "masteryType": "Melee",
  "masteryTier": 1
}
```

**`type: "LEAGUES_RELIC"`** (Relic Unlocked)
```json
{
  "relic": "Production Prodigy",
  "tier": 1,
  "requiredPoints": 0,
  "totalPoints": 20, // Player's current league points
  "pointsUntilNextTier": 480 // Optional
}
```

**`type: "LEAGUES_TASK"`** (Task Completed)
```json
{
  "taskName": "Pickpocket a Citizen",
  "difficulty": "EASY",
  "taskPoints": 10,
  "totalPoints": 30, // Player's current league points
  "tasksCompleted": 3,
  "tasksUntilNextArea": 57, // Optional
  "pointsUntilNextRelic": 470, // Optional
  "pointsUntilNextTrophy": 2470, // Optional
  "earnedTrophy": "Bronze" // Optional, if this task unlocked a trophy
}
```

---

### `type: "CHAT"`
Signifies a chat message matching a user-defined pattern.

**`extra` Object Example:**
```json
{
  "type": "GAMEMESSAGE", // Type of chat message, see RuneLite's ChatMessageType enum
  "message": "You've been playing for a while, consider taking a break from your screen.",
  "source": null, // Name of the player if it's a player chat type, else null or event source
  "clanTitle": null // Optional: { "id": integer, "name": string } for clan-related chats
}
```
**Key `extra` Fields:**
*   `type`: The type of chat message (e.g., "PUBLICCHAT", "GAMEMESSAGE", "CLAN_CHAT"). See RuneLite's `ChatMessageType` for all values.
*   `message`: The content of the chat message.
*   `source`: If the chat message is from a player (e.g., public chat), this is their name. Otherwise, it might be null or indicate an event source (e.g., "CommandExecuted").
*   `clanTitle`: An object with `id` (integer) and `name` (string) of the sender's clan rank/title, if applicable to the chat type.

---

### `type: "EXTERNAL_PLUGIN"`
Signifies a webhook triggered by another RuneLite plugin via Dink.

**`extra` Object Example:**
```json
{
  "sourcePlugin": "My Awesome Plugin",
  "metadata": { // Custom data provided by the source plugin
    "custom_key": "custom_value",
    "another_metric": 123
  }
}
```
**Key `extra` Fields:**
*   `sourcePlugin`: The name of the RuneLite plugin that requested this webhook.
*   `metadata`: An object containing custom data sent by the `sourcePlugin`. Its structure is entirely dependent on that plugin and can be `null`.

---

### `type: "LOGIN"`
Special metadata event sent on login, summarizing character state. Delayed by ~5 seconds after login.

**`extra` Object Example (partial):**
```json
{
  "world": 338,
  "collectionLog": {"completed": 651, "total": 1477},
  "combatAchievementPoints": {"completed": 503, "total": 2005},
  "achievementDiary": {"completed": 42, "total": 48},
  "achievementDiaryTasks": {"completed": 477, "total": 492},
  "barbarianAssault": {"highGambleCount": 0},
  "skills": {
    "totalExperience": 346380298,
    "totalLevel": 2164,
    "levels": {"Attack": 107, /* ... */},
    "experience": {"Attack": 30696420, /* ... */}
  },
  "questCount": {"completed": 156, "total": 158},
  "questPoints": {"completed": 296, "total": 300},
  "slayer": {"points": 2204, "streak": 1074},
  "pets": [
    {"itemId": 11995, "name": "Pet chaos elemental"}
  ]
}
```
**Key `extra` Fields:**
*   Contains a snapshot of various player stats: `world`, `collectionLog` progress, `combatAchievementPoints`, `achievementDiary` progress (both tiers and tasks), `barbarianAssault` gamble count, detailed `skills` data (total XP, total level, individual skill levels and XP), `questCount` and `questPoints`, `slayer` points and streak, and a list of owned `pets` (`itemId`, `name`).
*   Note: Some data like `collectionLog` might be missing if certain in-game interfaces haven't been opened by the player recently. `pets` requires the base Chat Commands plugin to be enabled in RuneLite.

---

### `type: "LOGOUT"`
Signifies the player has logged out. The `extra` object for this type is typically empty or `null`.

**`extra` Object Example:**
```json
null
// or
{}
```

---

### `type: "TOA_UNIQUE"`
Signifies receiving a unique (purple) item from a Tombs of Amascut raid. This is a specialized event, distinct from the general "LOOT" event for the actual items.

**`extra` Object Example:**
```json
{
  "party": ["Player1", "Player2", "Player3"],
  "rewardPoints": 30000, // Points in the raid
  "raidLevel": 550,    // Invocation level
  "probability": 0.2    // Chance of unique at these points/level
}
```
**Key `extra` Fields:**
*   `party`: Array of player names in the raid party.
*   `rewardPoints`: Points accumulated in the Tombs of Amascut raid.
*   `raidLevel`: The invocation level of the raid.
*   `probability`: The calculated probability of receiving a unique item with the given `rewardPoints` and `raidLevel`.

---

## 4. Example Next.js API Route Handler

Below is a more complete example of a Next.js API route handler using `formidable` (for the Node.js runtime). This demonstrates how to structure your logic to handle different event types.

```javascript
// pages/api/dink-webhook.js
import formidable from 'formidable';

// Disable Next.js's default body parser for this route
export const config = {
  api: {
    bodyParser: false,
  },
};

export default async function handler(req, res) {
  if (req.method === 'POST') {
    const form = formidable({});

    try {
      const [fields, files] = await form.parse(req);

      const payloadJsonString = fields.payload_json?.[0];
      const imageFile = files.file?.[0]; // This is a formidable File object

      if (!payloadJsonString) {
        return res.status(400).json({ error: 'Missing payload_json' });
      }

      const webhookData = JSON.parse(payloadJsonString);
      const eventType = webhookData.type;
      const eventDetails = webhookData.extra;
      const playerName = webhookData.playerName;

      console.log(`Received webhook for player: ${playerName}, type: ${eventType}`);

      // Example: Storing data or sending notifications
      // This is where you would integrate with your database, push to a client via WebSockets, etc.
      // For demonstration, we'll just log some details.

      switch (eventType) {
        case 'LOOT':
          console.log(`${playerName} received loot: ${eventDetails.items.map(item => `${item.quantity}x ${item.name}`).join(', ')} from ${eventDetails.source}`);
          if (eventDetails.killCount) {
            console.log(`Kill count: ${eventDetails.killCount}`);
          }
          // Your logic to display loot in your app
          break;

        case 'LEVEL':
          const skillsLeveled = Object.keys(eventDetails.levelledSkills)
            .map(skill => `${skill} to ${eventDetails.levelledSkills[skill]}`)
            .join(', ');
          console.log(`${playerName} leveled up: ${skillsLeveled}. Total level: ${eventDetails.allSkills.totalLevel || webhookData.skills?.totalLevel}`); // Fallback for older LOGIN data structures if misused
          // Your logic to display level ups
          break;

        case 'DEATH':
          let deathMessage = `${playerName} died.`;
          if (eventDetails.isPvp && eventDetails.killerName) {
            deathMessage = `${playerName} was PKed by ${eventDetails.killerName}.`;
          } else if (eventDetails.killerName) {
            deathMessage = `${playerName} was killed by ${eventDetails.killerName}.`;
          }
          deathMessage += ` Value lost: ${eventDetails.valueLost} GP.`;
          console.log(deathMessage);
          // Your logic to display death info
          break;

        case 'COLLECTION':
          console.log(`${playerName} added ${eventDetails.itemName} to their collection log! (${eventDetails.completedEntries}/${eventDetails.totalEntries})`);
          // Your logic for collection log updates
          break;

        case 'SLAYER':
          console.log(`${playerName} completed slayer task: ${eventDetails.slayerTask}. Points: ${eventDetails.slayerPoints}. Tasks done: ${eventDetails.slayerCompleted}.`);
          // Your logic for slayer updates
          break;

        case 'LOGIN':
          console.log(`${playerName} logged in on world ${eventDetails.world}. Total level: ${eventDetails.skills?.totalLevel}`);
          // Your logic for login events (e.g., update online status)
          break;

        // Add more cases for other event types you want to handle...
        // case 'QUEST':
        // case 'CLUE':
        // case 'KILL_COUNT':
        // ... etc.

        default:
          console.log(`Received unhandled event type: ${eventType}`);
      }

      if (imageFile) {
        console.log(`Screenshot received: ${imageFile.originalFilename}, size: ${imageFile.size} bytes, temp path: ${imageFile.filepath}`);
        // Here you might process the image:
        // - Save it to a persistent storage (e.g., S3, local filesystem if not serverless)
        // - Associate it with the event data
        // Example: await saveImageToCloud(imageFile.filepath, `screenshots/${playerName}/${eventType}-${Date.now()}`);
      }

      res.status(200).json({ message: 'Webhook processed successfully' });

    } catch (error) {
      if (error instanceof SyntaxError) {
        console.error('Error parsing JSON payload:', error);
        return res.status(400).json({ error: 'Invalid JSON payload' });
      }
      console.error('Error processing webhook:', error);
      res.status(500).json({ error: 'Internal server error while processing webhook' });
    }
  } else {
    res.setHeader('Allow', ['POST']);
    res.status(405).end(`Method ${req.method} Not Allowed`);
  }
}

// Example utility function (you'd implement this based on your needs)
// async function saveImageToCloud(localPath, cloudPath) {
//   //  Your logic to upload file from localPath to cloudPath
//   console.log(`Simulating upload of ${localPath} to ${cloudPath}`);
//   //  e.g., using aws-sdk, @google-cloud/storage, etc.
//   //  Remember to handle cleanup of the temporary file if necessary, though formidable might do this.
// }
```

This example provides a basic routing mechanism using a `switch` statement. For more complex applications, you might consider a more sophisticated handler registration pattern. Remember to implement the actual logic for what your application should *do* with this information within each case block.

---

## 5. Security and Best Practices

When implementing your webhook handler, consider the following:

*   **Webhook URL Secrecy:** Your webhook endpoint URL is a secret. Anyone who has it can send data to your server. Keep it confidential. If it leaks, you may need to change it and update your Dink configuration.
*   **No Built-in Signature Verification (Currently):** As of the `docs/json-examples.md` documentation, Dink webhooks do not explicitly mention a system for verifying request signatures (e.g., like a `X-Hub-Signature` header HMAC signed with a shared secret).
    *   **Consideration:** If this is a critical requirement, you might explore options like:
        *   Only allowing requests from specific IP addresses (if Dink server IPs are static and known, though this is often not feasible or reliable for client-side plugins).
        *   Implementing a custom shared secret that the Dink plugin user could configure (if Dink plugin allows custom headers) and your server could verify. This would be an advanced setup.
        *   For most use cases with Dink (which originates from a client-side game plugin), the primary "trust" comes from the `dinkAccountHash` and the fact that the user configured their client to point to your URL.
*   **Input Validation:**
    *   Always validate the `eventType` and expect the corresponding `extra` structure.
    *   Be cautious with data that could be unusually large or malformed, especially if it's being inserted into a database or displayed directly. Sanitize outputs if displaying data as HTML.
    *   The `content` field, being user-configurable, should especially be treated as untrusted if you decide to use it directly.
*   **Error Handling:**
    *   Wrap JSON parsing in `try...catch` blocks as malformed JSON will throw an error.
    *   Implement robust error logging for your webhook handler to diagnose issues.
    *   If an unknown `eventType` is received, log it and respond appropriately (e.g., `200 OK` to acknowledge receipt but note it was unhandled, or a `400 Bad Request` if you prefer stricter handling).
*   **Respond Quickly:**
    *   Webhook providers generally expect a quick response (e.g., within a few seconds) to acknowledge receipt. A common practice is to respond with a `200 OK` status as soon as the request is validated and basic parsing is done.
    *   If processing the webhook data takes significant time (e.g., multiple database calls, external API interactions, image manipulation), consider offloading the work to a background job queue (e.g., using BullMQ, Redis, or a serverless function) to ensure your API route responds swiftly. The API route would then just add the job to the queue.
*   **Idempotency (If Applicable):** Dink itself might retry sending webhooks if it encounters network issues. While not explicitly detailed as a guaranteed "at-least-once" delivery with retry mechanisms in the provided docs, it's a good practice to design your processing logic to be idempotent if possible. This means processing the same event multiple times should not result in duplicated data or incorrect states in your application. Using the `dinkAccountHash` and potentially a unique ID from the event data (if available and consistent across retries) can help. However, Dink events don't seem to have unique event IDs in the current JSON structure, so you might need to derive a unique signature based on content and timestamp if this is a concern.
*   **HTTPS:** Always use HTTPS for your webhook endpoint to protect the data in transit. Next.js deployments on platforms like Vercel or Netlify handle this automatically.
*   **Resource Limits:** Be mindful of resource limits if processing involves file uploads (screenshots). Ensure your server/platform can handle the expected file sizes and processing load. Formidable's default is to stream files to disk, so ensure you have disk space and permissions, or configure it to handle files in memory if they are small and your environment supports it.

By following these practices, you can create a more robust and secure Dink webhook integration.

---
(End of Documentation)
---
