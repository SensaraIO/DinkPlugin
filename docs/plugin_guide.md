# Processing Webhook Data from Dink Plugin

This document outlines how to receive and process data sent from the Dink RuneLite plugin to your webhook URL. The plugin sends notifications for various in-game events, including loot drops and Grand Exchange transactions.

## 1. Webhook Data Reception

All notifications from the Dink plugin are sent to your configured webhook URL via an HTTP `POST` request. The structure of this request is as follows:

*   **`Content-Type: multipart/form-data`**: The request body is formatted as multipart form data. This is important because notifications can include both JSON data and an optional screenshot image.
*   **Form Fields**:
    *   **`payload_json`**: This form field contains a JSON string. This string is the primary data payload holding all the information about the event. Your server will need to extract this string and then parse it as JSON.
    *   **`file` (Optional)**: This form field may contain an image file (e.g., `image/png` or `image/jpeg`). This is typically a screenshot of the game event. Your server can choose to process or ignore this file.

**Example (Conceptual):**

A typical HTTP request might look something like this (simplified):

```http
POST /your-webhook-endpoint HTTP/1.1
Host: yourserver.com
Content-Type: multipart/form-data; boundary=----WebKitFormBoundary7MA4YWxkTrZu0gW

------WebKitFormBoundary7MA4YWxkTrZu0gW
Content-Disposition: form-data; name="payload_json"

{ ... JSON data as a string ... }
------WebKitFormBoundary7MA4YWxkTrZu0gW
Content-Disposition: form-data; name="file"; filename="screenshot.png"
Content-Type: image/png

... binary image data ...
------WebKitFormBoundary7MA4YWxkTrZu0gW--
```

Your server-side application will need to be capable of parsing `multipart/form-data` to access the `payload_json` and the optional `file`. Most modern web frameworks have built-in utilities or libraries for this (e.g., Multer for Node.js/Express, `request.form` in Flask/Django, etc.).

---

## 2. Core JSON Structure (`payload_json`)

Once you've extracted the string from the `payload_json` form field, you'll parse it into a JSON object. All notification types share a common base structure.

**Common Fields (Present in every notification):**

*   **`type` (String):** This is a crucial field that identifies the kind of event being notified. Examples include `"LOOT"`, `"GRAND_EXCHANGE"`, `"DEATH"`, `"LEVEL"`, etc. You'll use this field to determine how to interpret the `extra` object.
*   **`extra` (Object):** This object contains the specific data relevant to the `type` of notification. Its structure varies significantly based on the event type. For third-party integrations, this is often the most important part of the payload.
*   **`playerName` (String):** The RuneScape Name (RSN) of the player who triggered the event.
    *   Example: `"your rsn"`
*   **`accountType` (String):** The player's account type.
    *   Possible values: `"NORMAL"`, `"IRONMAN"`, `"HARDCORE_IRONMAN"`, `"ULTIMATE_IRONMAN"`, `"GROUP_IRONMAN"`, `"HARDCORE_GROUP_IRONMAN"`. (Note: For Leagues events, this will be `"IRONMAN"` regardless of the player's main game account type).
*   **`seasonalWorld` (Boolean):** Indicates if the event occurred on a seasonal world (e.g., Deadman Mode, Leagues).
    *   Possible values: `true`, `false`.
*   **`dinkAccountHash` (String):** A unique, anonymized hash identifying the Dink plugin user. This can be used to correlate events from the same user without revealing their actual account details.
    *   Example: `"abcdefghijklmnopqrstuvwxyz1234abcdefghijklmnopqrstuvwxyz"`
*   **`content` (String):** A user-configurable message template that gets populated with event details. While present, it's generally recommended for third-party services to parse the structured data in `extra` rather than trying to parse this string, as its format can be changed by the user.
    *   Example: `"%USERNAME% has looted: %LOOT% From: %SOURCE%"`
*   **`embeds` (Array):** If the user has enabled the "Use Rich Embeds" advanced setting, this array will be populated according to Discord's embed structure. If not, it will usually be an empty array. Similar to `content`, relying on `extra` is more robust for data processing.

**Conditionally Present Fields (May or may not be present):**

These fields are only included in the JSON payload under certain conditions or if specific plugin settings are enabled:

*   **`clanName` (String):** The name of the player's clan.
    *   *Condition:* Player is in a clan AND the "Send Clan Name" advanced setting is enabled.
    *   Example: `"Dink QA"`
*   **`groupIronClanName` (String):** The name of the player's Group Ironman clan.
    *   *Condition:* Player is a Group Ironman AND the "Send GIM Clan Name" advanced setting is enabled.
    *   Example: `"Dink QA"`
*   **`discordUser` (Object):** Information about the user's Discord profile.
    *   *Condition:* Discord desktop client is running AND the "Send Discord Profile" advanced setting is enabled.
    *   Structure:
        ```json
        {
          "id": "012345678910111213",
          "name": "Gamer",
          "avatarHash": "abc123def345abc123def345abc123de"
        }
        ```
*   **`world` (Integer):** The game world number.
    *   *Condition:* The "Include Location" advanced setting is enabled (default: true).
    *   Example: `518`
*   **`regionId` (Integer):** The ID of the player's current map region.
    *   *Condition:* The "Include Location" advanced setting is enabled (default: true).
    *   Example: `12850`

**General JSON Example (Illustrative):**

```json
{
  "type": "NOTIFICATION_TYPE",
  "extra": {
    // ... type-specific data goes here ...
  },
  "playerName": "your rsn",
  "accountType": "NORMAL",
  "seasonalWorld": false,
  "dinkAccountHash": "abcdefghijklmnopqrstuvwxyz1234abcdefghijklmnopqrstuvwxyz",
  "content": "User-defined message about the event",
  "embeds": [],
  // Conditionally present fields might appear here:
  // "clanName": "Some Clan",
  // "world": 301
}
```

**Key Takeaway for Processing:**

1.  Parse the `payload_json` string.
2.  Check the value of the `type` field.
3.  Based on the `type`, access and process the fields within the `extra` object.

---

## 3. Loot Notification Details (`type: "LOOT"`)

When the `type` field in the core JSON structure is `"LOOT"`, the `extra` object will contain information about items looted by the player.

**`extra` Object Structure for Loot:**

*   **`items` (Array of Objects):** A list of all items included in this loot notification. Each object in the array represents a distinct item stack.
    *   **`id` (Integer):** The in-game Item ID.
        *   Example: `1234`
    *   **`quantity` (Integer):** The number of this item looted.
        *   Example: `1`
    *   **`priceEach` (Integer):** The value of a single unit of this item, in GP. This is typically the Grand Exchange price if available and the user has not disabled the "Use actively traded price" setting in RuneLite; otherwise, it's the item's store/alchemy price.
        *   Example: `42069`
    *   **`name` (String):** The in-game name of the item.
        *   Example: `"Some item"`
    *   **`criteria` (Array of Strings):** Indicates why this item was included in the notification (e.g., it met a value threshold, it's rare, etc.).
        *   Possible values correspond to the `LootCriteria` enum in the Dink plugin. Examples: `"VALUE"`, `"RARITY"`, `"UNIQUE"`.
        *   Example: `["VALUE"]`
    *   **`rarity` (Number or `null`):** The drop rarity of the item from the specific source, if known and applicable. This is often a fractional value (e.g., `0.001` for 1/1000). It will be `null` if the rarity isn't tracked, if the item always drops, or for non-NPC loot. This data is imperfectly scraped from the OSRS Wiki.
        *   Example: `0.001` or `null`
*   **`source` (String):** The name of the NPC, event, or activity that yielded the loot.
    *   Examples: `"Tombs of Amascut"`, `"General Graardor"`, `"Clue Scroll (Master)"`, `"Pickpocketing Goblins"`
*   **`party` (Array of Strings, Optional):** If the loot is from a group activity (specifically raids: Chambers of Xeric, Tombs of Amascut, Theatre of Blood), this array will contain the RSNs of the players in the party. Otherwise, this field might be absent or an empty array.
    *   Example: `["%USERNAME%", "another RSN", "yet another RSN"]`
*   **`category` (String):** The type of the loot source. This corresponds to RuneLite's `LootRecordType` enum.
    *   Possible values include: `"NPC"`, `"PLAYER"`, `"EVENT"` (e.g., clue scrolls, supply chests), `"PICKPOCKET"`, `"UNKNOWN"`, etc.
    *   Example: `"EVENT"`
*   **`killCount` (Integer, Optional):** The player's kill count for the specific `source` (if it's an NPC or certain events like Chambers of Xeric). This is only specified if the base RuneLite "Loot Tracker" plugin is enabled by the user.
    *   Example: `60`
*   **`rarestProbability` (Number, Optional):** The probability of obtaining the rarest item within this particular loot drop. This is typically present when there's a unique or very rare item in the `items` list.
    *   Example: `0.001`
*   **`npcId` (Integer, Optional):** The ID of the NPC, if the `category` is `"NPC"` or `"PICKPOCKET"`.
    *   Example: `2215` (for General Graardor)

**Example `extra` Object for Loot:**

```json
{
  "items": [
    {
      "id": 11832, // Armadyl chestplate ID
      "quantity": 1,
      "priceEach": 25000000, // Example price
      "name": "Armadyl chestplate",
      "criteria": ["RARITY", "VALUE"],
      "rarity": 0.001953125 // Example rarity (1/512)
    },
    {
      "id": 526, // Bones ID
      "quantity": 1,
      "priceEach": 100,
      "name": "Bones",
      "criteria": [], // Might be empty if not meeting specific criteria but part of the drop
      "rarity": null
    }
  ],
  "source": "Kree'arra",
  "party": [], // Not a raid, so party might be empty or absent
  "category": "NPC",
  "killCount": 152,
  "rarestProbability": 0.001953125,
  "npcId": 3162 // Kree'arra's NPC ID
}
```

**Processing Considerations for Loot:**

*   Iterate through the `items` array to process each item individually.
*   Use the `source` and `category` to understand the context of the loot.
*   The `killCount` can be useful for tracking boss progression.
*   `priceEach` can be summed with `quantity` for all items to get the total value of the loot drop.
*   The `rarity` and `rarestProbability` fields can be used to highlight exceptionally rare drops.

You can find more information about `LootRecordType` [here](https://github.com/runelite/api.runelite.net/blob/master/http-api/src/main/java/net/runelite/http/api/loottracker/LootRecordType.java) and `LootCriteria` [here](https://github.com/pajlads/DinkPlugin/blob/master/src/main/java/dinkplugin/domain/LootCriteria.java).

---

## 4. Grand Exchange Notification Details (`type: "GRAND_EXCHANGE"`)

When the `type` field in the core JSON structure is `"GRAND_EXCHANGE"`, the `extra` object will contain information about a player's Grand Exchange offer that has been updated (e.g., bought, sold, cancelled).

**`extra` Object Structure for Grand Exchange:**

*   **`slot` (Integer):** The Grand Exchange slot number for this offer. This is 1-indexed.
    *   Values: 1 to 8 for members, 1 to 3 for Free-to-Play.
    *   Example: `1`
*   **`status` (String):** The current state of the Grand Exchange offer. This corresponds to RuneLite's `GrandExchangeOfferState` enum.
    *   Common values:
        *   `"EMPTY"`: Slot is empty (less likely for a notification, usually means cancelled and collected).
        *   `"CANCELLED_BUY"`: A buy offer was cancelled.
        *   `"CANCELLED_SELL"`: A sell offer was cancelled.
        *   `"BUYING"`: A buy offer is in progress (partially filled or just placed).
        *   `"SELLING"`: A sell offer is in progress (partially filled or just placed).
        *   `"BOUGHT"`: A buy offer has been completed.
        *   `"SOLD"`: A sell offer has been completed.
    *   Example: `"SOLD"`
*   **`item` (Object):** Details about the item involved in the GE offer.
    *   **`id` (Integer):** The in-game Item ID.
        *   Example: `314` (Feather)
    *   **`quantity` (Integer):** The number of items that were part of this specific transaction update (e.g., if an offer for 1000 items sells 200, quantity here would be 200 for that notification). For a fully completed offer, this will match `targetQuantity`.
        *   Example: `2`
    *   **`priceEach` (Integer):** The actual price per unit at which this part of the transaction occurred.
        *   Example: `3`
    *   **`name` (String):** The in-game name of the item.
        *   Example: `"Feather"`
*   **`marketPrice` (Integer):** The item's current market price (actively traded price) on the Grand Exchange at the time of the notification, in GP.
    *   Example: `2`
*   **`targetPrice` (Integer):** The price per unit that the player originally set for the offer, in GP.
    *   Example: `3`
*   **`targetQuantity` (Integer):** The total quantity of items the player originally listed in the offer.
    *   Example: `2`
*   **`sellerTax` (Integer):** The amount of GP deducted as tax if the offer was a sale (`status: "SOLD"`). This is 0 for buy offers or incomplete/cancelled sell offers. The tax is 1% of the total sale value, rounded down.
    *   Example: `0` (if not sold or a buy) or a positive integer if sold.

**Example `extra` Object for Grand Exchange:**

```json
{
  "slot": 1,
  "status": "SOLD", // Or "BOUGHT", "CANCELLED_SELL", etc.
  "item": {
    "id": 314,
    "quantity": 2,     // How many were involved in this update
    "priceEach": 3,    // Actual transaction price per item
    "name": "Feather"
  },
  "marketPrice": 2,      // Current GE guide price
  "targetPrice": 3,      // Price player set in offer
  "targetQuantity": 2, // Total quantity player listed in offer
  "sellerTax": 0       // Tax if it was a sale (1% of item.quantity * item.priceEach)
}
```

**Processing Considerations for Grand Exchange:**

*   The `status` field is key to understanding what happened (item bought, sold, offer cancelled).
*   `item.quantity` and `item.priceEach` tell you the specifics of the transaction that just occurred.
*   `targetQuantity` and `targetPrice` provide the context of the original offer.
*   For `"SOLD"` statuses, `sellerTax` indicates the GE tax paid. The total GP received by the seller would be (`item.quantity` * `item.priceEach`) - `sellerTax`.
*   For `"BOUGHT"` statuses, the total GP spent would be `item.quantity` * `item.priceEach`.
*   An offer might generate multiple notifications if it's filled partially over time.

You can find the Javadoc for `GrandExchangeOfferState` [here](https://static.runelite.net/api/runelite-api/net/runelite/api/GrandExchangeOfferState.html) for a complete list of possible statuses.

---

## 5. General Processing Advice

Here are some general tips and best practices for building a server application to consume webhook notifications from the Dink plugin:

*   **Use Standard Libraries:**
    *   Leverage well-tested HTTP server libraries or frameworks for your chosen programming language (e.g., Express.js for Node.js, Flask/Django for Python, Spring Boot for Java, Gin for Go). These will handle the low-level complexities of HTTP requests.
    *   Utilize libraries specifically designed for parsing `multipart/form-data`. Most web frameworks have this capability built-in or offer official/recommended middleware (e.g., `multer` for Express, `request.files` in Flask). This will make accessing `payload_json` and the optional `file` straightforward.

*   **JSON Parsing:**
    *   Once you have the string content of the `payload_json` field, use your language's standard JSON parsing library (e.g., `JSON.parse()` in JavaScript, `json.loads()` in Python, Jackson/Gson in Java) to convert it into a native object or dictionary.

*   **Event Dispatching:**
    *   The `type` field in the parsed JSON is your primary dispatcher. Use a `switch` statement (if your language supports it well for strings) or a series of `if-else if` blocks to route the processing logic based on the notification type.
    ```javascript
    // Example in JavaScript
    const jsonData = JSON.parse(payloadJsonString);
    switch (jsonData.type) {
      case "LOOT":
        handleLootNotification(jsonData.extra, jsonData.playerName);
        break;
      case "GRAND_EXCHANGE":
        handleGrandExchangeNotification(jsonData.extra, jsonData.playerName);
        break;
      case "LEVEL":
        // handleLevelNotification(jsonData.extra, jsonData.playerName);
        break;
      // ... other cases
      default:
        console.log("Received unknown notification type:", jsonData.type);
    }
    ```

*   **Data Validation and Error Handling:**
    *   While the `json-examples.md` provides a good contract, it's wise to implement basic validation. Check for the presence of key fields you expect, especially within the `extra` object for the types you care about.
    *   Handle potential parsing errors (e.g., malformed JSON in `payload_json`, though unlikely from the plugin itself).
    *   Log errors gracefully. If your webhook endpoint encounters an issue, returning an appropriate HTTP error code (e.g., `400 Bad Request` for invalid data, `500 Internal Server Error` for server-side issues) can be helpful for debugging, though the plugin might not do anything specific with these responses.
    *   Consider that some fields are optional (e.g., `killCount` in loot, `party` in loot). Your code should be resilient to their absence if they are not critical for your use case.

*   **Security Considerations:**
    *   If your webhook URL is publicly accessible, consider implementing a mechanism to verify that incoming requests are genuinely from the Dink plugin or authorized users. This could be a secret token shared between the plugin configuration and your server, passed as a query parameter or a custom header (though Dink doesn't explicitly support adding custom headers to webhook requests out-of-the-box beyond the standard `multipart/form-data` structure). *Currently, the primary way to identify a user is via `dinkAccountHash`.*
    *   Be mindful of rate limiting if you expect a high volume of notifications or want to protect against potential (though unlikely) abuse.

*   **Asynchronous Processing:**
    *   For notifications that might trigger time-consuming tasks on your server (e.g., database writes, external API calls), consider processing them asynchronously. Your webhook endpoint could quickly acknowledge the request (e.g., send a `200 OK` or `202 Accepted` response) and then pass the data to a background worker or message queue. This prevents the Dink plugin from timing out while waiting for your server.

*   **Idempotency (If Applicable):**
    *   In distributed systems, webhook events can sometimes be delivered more than once. If the actions your server takes are not naturally idempotent (i.e., performing them multiple times has a different effect than performing them once), you might need to implement a way to detect and discard duplicate notifications. The `dinkAccountHash` along with a timestamp or a unique event identifier (if one were available in the payload, which is not explicitly the case here beyond the content itself) could be used for this. *Given the current payload, true idempotency might be tricky without deriving a unique key from the event content.*

*   **Refer to `json-examples.md`:**
    *   Keep the `docs/json-examples.md` document handy. It's the source of truth for the data structures.

By following these guidelines, you can build a robust and reliable server application to handle webhook notifications from the Dink plugin.
