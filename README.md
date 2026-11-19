# How to Build a Persistent Wishlist in FlutterFlow with Shopify Storefront API

Adding a wishlist is one of the highest-leverage features you can build into an e-commerce mobile app. By combining **FlutterFlow's persisted App State** with **Shopify's Storefront GraphQL API**, shoppers can save products instantly—even while browsing as a guest—without slowing down your app or requiring extra backend databases.

---

## Why a Wishlist Matters (Benefits & Importance)

* **Reduces Cart Abandonment & Friction:** Shoppers often browse before they are ready to buy. A wishlist gives hesitant buyers a low-commitment way to bookmark items instead of losing track of them or cluttering their shopping cart.
* **Boosts Return Visits & Conversion Rates:** Saved items act as a personal shopping list that brings users back into the app, turning casual window-shoppers into repeat buyers.
* **Works Immediately for Guest Users:** Storing product IDs locally in persisted App State means users do not need to create an account or log in just to save products, eliminating early onboarding drop-off.
* **Zero Extra Backend Cost & Fast Batch Hydration:** Instead of storing full product objects locally (which go stale when prices or stock change), saving lightweight Shopify Product GIDs and hydrating them on demand via GraphQL `nodes(ids: ...)` keeps prices, images, and availability live in a single API call.

---

## Step 1: Create a Persisted App State Variable

1. Open **App Values** in the left navigation bar and select **App State**.
2. Click **+ Add State Variable** and configure:
* **Field Name:** `wishlistProductIds`
* **Type:** `String`
* **Is List:** Toggle **ON**
* **Persisted:** Toggle **ON** *(saves the list to local device storage so saved items remain after the app closes)*
* **Default Value:** Leave empty (`[]`).



---

## Step 2: Create the Shopify Storefront GraphQL API Call

Use Shopify's `nodes` query to fetch live product data for every saved GID in a single batch request.

1. Go to **API Calls** in FlutterFlow and click **+ Add** → **Create API Call**.
2. Configure the request:
* **API Call Name:** `GetWishlistProducts`
* **Method:** `POST`
* **API URL:** `https://<your-store-domain>/api/2026-04/graphql.json` *(use your active Storefront API version)*


3. Add the required **Headers**:
* `Content-Type`: `application/json`
* `X-Shopify-Storefront-Access-Token`: `<your-storefront-access-token>`


4. Define the **Variables**:
* **Name:** `ids` | **Type:** `String` | **Is List:** Toggle **ON**


5. Set **Body** format to **JSON** and paste the GraphQL query (keep `<ids>` outside quotes and brackets so FlutterFlow passes the JSON array directly):
```json
{
  "query": "query GetWishlistProducts($ids: [ID!]!) { nodes(ids: $ids) { ... on Product { id title handle featuredImage { url altText } priceRange { minVariantPrice { amount currencyCode } } } } }",
  "variables": {
    "ids": <ids>
  }
}

```


6. Go to **Response & Test**, enter a sample JSON array of valid product GIDs in `ids` (e.g., `["gid://shopify/Product/1234567890"]`), check **Include**, and click **Test API Call**.
7. Under **JSON Paths**, click **+ Add JSON Path** and save:
* **Name:** `wishlistProducts`
* **JSON Path:** `$.data.nodes`



---

## Step 3: Add the Interactive Heart Toggle to Your Product Card & Product Details Page

Using a **`Stack`** with two `IconButton`s (`Liked` and `Unliked`) controlled by **Conditional Visibility** gives you instant UI feedback across every screen.

1. Place a **`Stack`** in the top-right corner of your product image (or header bar) containing two `IconButton` widgets:
* **`Liked` Button:** Filled heart icon (`Icons.favorite_rounded`, red accent `#EF4444`).
* **`Unliked` Button:** Outlined heart icon (`Icons.favorite_border_rounded`).


2. Configure **Conditional Visibility** on each button:
* **On `Liked`:** Turn **Conditional Visibility** ON → **Single Condition** → **App State (`wishlistProductIds`)** → **Available Options: `List Contains Item**` → **Item:** `<productId>` → **Equal To** → **`True`**.
* **On `Unliked`:** Turn **Conditional Visibility** ON → **Single Condition** → **App State (`wishlistProductIds`)** → **Available Options: `List Contains Item**` → **Item:** `<productId>` → **Equal To** → **`False`**.


3. Configure the **`On Tap`** actions:
* **On `Liked` (Remove):** **Update App State** → `wishlistProductIds` → **Remove from List** → `<productId>` → **Rebuild All Pages**.
* **On `Unliked` (Add):** **Update App State** → `wishlistProductIds` → **Add to List** → `<productId>` → **Rebuild All Pages**.



---

## Step 4: Build the Wishlist Screen (`WishlistProducts`)

1. **Handle the Empty State First:**
* Add a **`ConditionalBuilder`** as the main body container.
* Set the condition to **App State (`wishlistProductIds`) → `Is Set and Not Empty**`.
* **Else (False) Branch:** Display an empty-state `Column` with a heart icon, title (**"Your wishlist is empty"**), subtitle (**"Save items you love while browsing to easily find and buy them anytime"**), and a button to browse products.


2. **Attach the API Query to Your GridView (`If (True)` Branch):**
* Inside the **`If (True)`** branch, add a 2-column **`GridView`**.
* Attach a **Backend Query → API Call (`GetWishlistProducts`)** directly to the `GridView` (this automatically manages loading states so your UI never renders before data arrives), passing **App State (`wishlistProductIds`)** to `ids`.


3. **Generate Dynamic Children & Guard Against Deleted Products:**
* Open **Generate Dynamic Children** on the `GridView`, name the variable `productItem`, and bind it to the API response JSON Path `$.data.nodes`.
* If a saved product is ever deleted or unpublished in Shopify, Shopify's `nodes` query returns `null` at that index. Wrap your `ProductCard` inside a **`ConditionalBuilder`** checking `productItem Item → JSON Path ($.id) → Is Set and Not Empty` so deleted products do not crash the grid or leave blank card slots.


4. **Bind Your Product Card Parameters:**
* Map your reusable **`ProductCard`** fields from `productItem Item`:
* **Title:** `$.title`
* **Image URL:** `$.featuredImage.url`
* **Price:** `$.priceRange.minVariantPrice.amount`
* **Product ID:** `$.id`
* **Product Handle:** `$.handle`


* Set the card's **`On Tap`** action to navigate to your Product Details page, passing both `productId` and `productHandle`.



---

## Step 5: Surface the Wishlist in Navigation (Optional Polish)

* **Bottom Navigation Bar:** Include a **Wishlist** tab (`Icons.favorite_rounded` / `Icons.favorite_border_rounded`) in a standard 5-item bottom bar (`Home`, `Categories`, `Wishlist`, `My Cart`, `Profile`) with a live badge showing `wishlistProductIds.length` whenever the count is greater than `0`.
* **Profile Menu:** Add a **My Wishlist** shortcut row in your account/profile menu so users can always find their saved items quickly.
