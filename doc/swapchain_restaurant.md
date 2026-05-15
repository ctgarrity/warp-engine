# The Swapchain, Explained as a Restaurant

A narrative explanation of the Vulkan swapchain synchronization used in this
sample. Everything here maps back to real Vulkan objects and functions -- see
the [translation table](#translation-table) at the end of the document for the
one-to-one mapping -- but the story alone is self-contained.

## Who's who

- 🧑‍🍳 **One cook.** That's the GPU. He works fast but only on one ticket at a
  time, and he never puts a ticket down -- once he picks it up, he finishes
  it before starting the next.
- 🍽️ **Three plates.** Every dish that goes out has to be on one of these
  three -- no extras, no substitutes.
- 🤵 **Two waiters.** They're the only ones allowed to take orders and talk
  to the cook.
- 🧽 **The dish station.** The only person who decides which plate is
  available at any moment. Waiters ask her for a plate; they don't pick. She
  also collects plates coming back from the dining room and puts them back
  in rotation.
- 🏃 **The runner.** Carries finished plates from the kitchen to the dining
  room, but only when the cook says the dish is done.
- 🍴 **The dining room.** Diners eat whatever plate a waiter brings out, at
  their own pace. When a diner is finished, the plate goes back to the dish
  station.

## Two sets of things belong to two different people

Each 🍽️ **plate** has a little hanger on the side that the cook clips a
🔖 *"this dish is done, ready to serve"* note to. That note follows the plate
-- not the waiter -- because the dish station can hand plates back in any
order, and the note needs to match the plate it's actually clipped to.

Each 🤵 **waiter** carries:
- 📓 Their own notepad -- a pad of blank order tickets they can write on.
- 🎟️ A *"a clean plate has been reserved for me"* slip that the dish station
  gives them each time they ask for a plate. This belongs to the waiter
  because each waiter manages their own cycle.
- 📇 A personal reminder card with a number on it -- "last time I worked, my
  ticket was #5."

There's also one big 🧾 **receipt spike** on the wall. Every time the cook
finishes cooking a ticket, he stabs it onto the spike and the number goes up
by one: 1, 2, 3, 4, 5... The number on the spike only ever increases.

## One service round, step by step

Let's follow **Waiter A** as she walks back onto the floor. Her reminder card
says "my last ticket was #3." The spike on the wall currently reads 4.

### Step 1 📇 Check the reminder card

"Am I safe to reuse my notepad and tools? I told the cook to stamp #3 when he
finished my last order. Has the spike reached 3?"

Yes -- the spike is already at 4. She's safe to start. (If the spike still
read 2, she'd stand and wait. That's the only moment she's ever deliberately
stuck.)

### Step 2 🍽️ Ask the dish station for a plate

She walks to the dish station: "I need a plate." **The station picks which
one** -- say plate #2 -- based on whatever is available and her own ordering
rules. The waiter doesn't choose and can't predict which plate she'll get;
this is important, because it means the waiter's identity and the plate
number are not correlated.

The station hands her the plate and says "here's your reservation slip -- the
plate will be clean when I hand this slip back to you." She pockets the slip
and walks on. She doesn't wait for the plate to actually be clean right now;
the slip is a promise that will be fulfilled later.

### Step 3 📝 Write the order

She sits at her station and fills in a fresh ticket from her notepad:
*"rotating triangle, compute-animated vertices, textured triangle, four
flashing dots, overlay the menu."* The cook isn't doing anything yet -- she's
just writing.

### Step 4 🧑‍🍳 Hand the ticket to the cook, with rules

"Here's the ticket. Before you actually start plating, wait for this
clean-plate slip of mine. When you're done, two things: clip the 'dish is
done' note to plate #2, AND stab the next number onto the spike -- that'll
be #5."

She also writes **5** on her reminder card and tucks it back in her apron.
That's her sticky note for the next round.

### Step 5 🏃 Queue a delivery with the runner

Important: the plate isn't actually cooked yet. Waiter A hasn't been
standing around waiting for the cook -- she handed him the ticket and kept
moving. So she doesn't carry the plate to the dining room, she drops a
**delivery order** with the runner: "take plate #2 out to the diners, but
wait for the 'dish is done' note on *this plate* before putting it down."

Two notes, two jobs:
- The runner is waiting on the *plate's* "dish is done" note (the one the
  cook will clip when he finishes).
- That's a different note than the waiter's "clean plate" reservation slip
  from step 2, which was about the plate becoming available in the first
  place.

### Step 6 🔄 Swap with the other waiter

Waiter A steps back. Waiter B walks forward. Same routine from step 1,
starting from her own reminder card.

## Meanwhile, in the kitchen

All of that happened on the waiters' side of the pass. The cook has been
working at his own pace the whole time. Every time he finishes a ticket, he
does two things, in order, and then immediately picks up the next ticket in
his queue:

1. **Clips the "dish is done" note to the correct plate.** That's what lets
   the runner actually serve it -- until this moment the runner's delivery
   order is parked, waiting for the note.
2. **Stabs the next number onto the receipt spike.** 1, then 2, then 3, 4,
   5... The number only ever goes up. This is what the waiters' reminder
   cards refer back to.

The cook never stops, never waits for waiters, never cares who ordered what.
He just cooks tickets and clips notes and stabs numbers. The waiters and the
runner check in asynchronously: they read whatever notes are on plates they
hold, and they glance at the spike when they need to reuse their own tools.
That's the whole synchronisation between the cook and the floor.

## The plate's journey home

A plate doesn't disappear after the runner delivers it. It has a whole
afterlife that closes the loop:

1. Diner gets the plate, eats at their own pace (a TV showing frames at 60
   Hz, say).
2. When the diner is done, the plate goes back to the dish station.
3. The dish station washes it and puts it back in rotation.
4. The next time a waiter walks up asking for a plate, the station may hand
   this one back out.

That's why the waiter's "clean plate" reservation slip is a promise rather
than instant: when the waiter asks for a plate, the plate she's assigned may
still be at a diner's table, or mid-wash, or already back in the stack. The
station issues the slip immediately (so the waiter isn't blocked) but the
slip only fires -- clipping back to "ready" -- once the plate is actually
clean and available.

And that's also why the dish station hands out plates in an **unpredictable
order**. It depends on when diners finish, how fast the wash goes, and how
many plates are in the stack. Over many service rounds you'll see patterns
(sometimes a plate keeps coming back to the same waiter, sometimes they
rotate cleanly) but nothing about the pattern is guaranteed -- which is
exactly why every per-plate note has to live with the plate.

## What happens when Waiter A comes back

Only two waiters, so A is up again quickly. Her card still says **5**. She
checks the spike.

- If the spike reads 5 or higher -- cook has finished -- she continues
  immediately.
- If the spike still reads 4 -- cook is still working on her last order --
  she stands and waits. That's fine; it means the GPU is busy, and blocking is
  the correct behaviour. Without this wait she'd reuse her notepad while the
  cook was still reading from it, and dinner would be a disaster.

## Why 3 plates and 2 waiters?

**Three plates, not two.** Imagine there are only two plates total. One
waiter grabs one and takes it to the cook. Meanwhile, a diner in the dining
room is still eating off the other plate. The second waiter walks up to
the dish station -- both plates are gone. She has to stand and wait for the
diner to finish. With three plates, even if one is at a diner's table, two
are still in circulation, so both waiters can keep the cook busy.

**Two waiters, not three.** A third waiter would let the CPU race even further
ahead of the cook. That sounds good -- more tickets pre-written! -- but it
means any order a diner places shows up on a plate three rounds later instead
of two. In an interactive restaurant (someone moving a slider, clicking a
button), that extra round of lag is visible. Two waiters is enough to keep
the cook busy without making diners wait.

**One cook, not two.** Single GPU queue. A bigger kitchen with an oven crew
and a grill crew could parallelise -- that's multi-queue Vulkan -- but this
restaurant keeps it simple: one cook, one ticket at a time, in order.

## The three critical rules, back in restaurant terms

1. 🎟️ **"Clean plate" slips belong to the waiter.** She asked, the station
   promised, she uses it on her turn.
2. 🔖 **"Dish is done" notes belong to the plate.** Plates can come back from
   the dining room in any order, and the note must match whichever plate is
   in your hand.
3. 🧾 **The receipt spike protects the waiter's tools.** One counter, only
   goes up. Each waiter's 📇 reminder card says the number she's waiting on.
   Reuse only after the cook has passed that number.

That's the whole loop. Cook cooks, waiters keep two orders moving, three
plates rotate between kitchen and dining room, and a single ever-increasing
number on the wall keeps everyone honest.

## Translation table

| In the restaurant | In Vulkan | Where it lives in the code |
| --- | --- | --- |
| 🧑‍🍳 Cook | GPU queue | `Context::m_queues[0].queue` |
| 🍽️ Plates | Swapchain images | `Swapchain::m_nextImages` (sized by `m_imageCount`) |
| 🤵 Waiters | In-flight slots | `Swapchain::m_inFlightSlots` (sized by `m_framesInFlight`), plus `MinimalLatest::m_frameData` |
| 🧽 Dish station | Swapchain / presentation engine | `VkSwapchainKHR` (owned by `Swapchain::m_swapChain`) |
| 🏃 Runner | Presentation engine's delivery side | `vkQueuePresentKHR` plumbing |
| 🍴 Dining room / diners | Display (compositor, monitor) | outside Vulkan's direct control |
| 🧾 Receipt spike | Timeline semaphore | `MinimalLatest::m_frameTimelineSemaphore` |
| 🔢 Receipt number on the spike | Timeline counter | `MinimalLatest::m_frameCounter` |
| 🎟️ "Clean plate" slip (per waiter) | `acquireSemaphore` binary semaphore | `Swapchain::InFlightSlot::acquireSemaphore` |
| 🔖 "Dish is done" note (per plate) | `presentSemaphore` binary semaphore | `Swapchain::SwapchainImage::presentSemaphore` |
| 📇 Waiter's reminder card (last ticket I handed out) | Timeline value the slot must wait for | `MinimalLatest::FrameData::lastSignalValue` |
| 📓 Waiter's notepad | Command pool + command buffer | `MinimalLatest::FrameData::cmdPool` / `cmdBuffer` |
| 🤵 Current waiter on the floor | Current in-flight slot index | `Swapchain::getFrameResourceIndex()` |
| 🍽️ Plate # in the waiter's hand | Current swapchain image index | `Swapchain::m_frameImageIndex` |
| 🍽️ Ask the dish station for a plate | `vkAcquireNextImageKHR` | `Swapchain::acquireNextImage` |
| 🧑‍🍳 Hand the ticket to the cook | `vkQueueSubmit2` | `MinimalLatest::endFrame` |
| 🏃 Send the plate to the dining room | `vkQueuePresentKHR` | `Swapchain::presentFrame` |
| 📇 Check the reminder card / wait for the spike | `vkWaitSemaphores` on the timeline | `MinimalLatest::prepareFrameResources` |

## Further reading

- [`README.md`](../README.md) -- the feature list, build instructions, and the
  technical version of the synchronization section (with mermaid diagrams).
- [`src/minimal_latest.cpp`](../src/minimal_latest.cpp) -- the actual code;
  `Swapchain`, `MinimalLatest::prepareFrameResources`, and
  `MinimalLatest::endFrame` are the three places this story plays out.
- Define `NVVK_SEMAPHORE_DEBUG` when building to print a one-line trace at
  each step of every frame -- you can watch the receipt numbers and plate
  indices live.
