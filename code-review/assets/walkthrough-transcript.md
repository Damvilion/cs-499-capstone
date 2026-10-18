# Travlr Getaways — Module Two Code Review

## 00:00:00 — Introduction

Personal introduction from Recordings/new.mp4. This recording has not been transcribed.

## 00:00:44.100 — How the app is put together

Let's start with app dot js. This is where Express connects the main parts of the application. Requests that start with slash api go to the API routes. The travel pages have their own route, and the public folder holds things like images and styles. Handlebars builds the pages that customers see. The Angular admin app talks to the same backend. So there is already some separation here. I don't need to start over or replace the whole design.

This file also handles a few common errors. If a request contains broken JSON, the server sends back a four hundred response. If something unexpected goes wrong, it sends a general server error instead of showing the full error to the person using the site. Those are useful protections, and I want to keep them when I make changes.

The part I'd clean up is the trip controller. Right now, it checks requests, applies trip rules, and talks to the database. I'd split those jobs into smaller pieces. The controller would handle the request and response. A service would handle the trip rules. A separate file would handle database calls. That way, changing a trip rule wouldn't mean digging through code for all three jobs. It should also make the tests easier to follow.

## 00:01:52.600 — What happens when someone saves a trip

Here is the save function in the Angular form. Before it sends anything, it checks whether the form is valid and whether a save is already running. That second check helps prevent someone from sending the same request twice by clicking Save again. If the form needs fixing, it marks the fields so the person can see the errors. Otherwise, it either updates an existing trip or adds a new one. After a successful save, it returns to the list and shows a message.

One detail I want to keep is the original trip code. Someone can edit the code in the form, so the app needs both values. The old code tells the server which trip to find. The new code is part of the update. For example, if I change a code from A to B, the server still has to find A first. The data service sends the request with the login token, and the backend passes it to the trip controller.

The form checks are helpful, but someone can send a request without using this form at all. The server needs its own checks. I'd test missing fields, bad values, duplicate codes, and attempts to update a trip that doesn't exist. I'd also test changing the code itself. The goal is to keep the current save flow while making sure those less common cases behave properly.

## 00:03:02.567 — Giving each part of the code one job

This update function shows why I'd split up the trip controller. It checks the trip code, picks fields from the request, checks for missing values, updates the database, and decides which response to send back. It also handles database errors. None of that is unusual for a small app, but it makes this one function responsible for a lot. As I add more trip rules, it will get harder to change one thing without affecting something else.

There are details here that already work well. The update asks Mongoose to return the changed trip, and it turns on validation for the update. I'd keep both settings. In the new layout, the controller would pass the request to the trip service. The service would check the trip rules and ask the database code to save the result. If a trip code is already taken, the controller would still send a clear conflict response. The person using the app shouldn't lose useful feedback just because I've moved some code.

I also don't want to overdo this. More files aren't automatically better. I'd start with the trip functions I'm actually changing and keep the new pieces small. The current code already lets tests supply a fake model, which is useful. I'd keep those tests running as I move things around, so I can tell whether I've changed behavior by mistake.

## 00:04:18.733 — Who should be allowed to change trips

The trip routes already require a login token for adding, editing, or deleting a trip. Reading trips is public. That makes sense for a travel site: anyone should be able to browse, but changing the catalog should be restricted. The problem is that registration is also public. A new account gets a token, and the token check lets that request continue. I don't see another check here that asks whether the account is actually an administrator.

So the missing question is, who gets permission to manage trips? Being logged in doesn't answer that by itself. I'd either restrict how admin accounts are created or add an admin role that the server checks before allowing changes. I'd settle that rule before writing the code. This finding comes from following the routes and login checks in the source. I haven't tried it against a live website, so I'm not claiming that I've tested an attack on a deployed app.

To verify the fix, I'd test three cases. A person with no token should be turned away. A logged-in person without admin permission should also be turned away. An approved admin should be able to reach the trip function. Those tests need to use the real permission checks. Hiding the Edit button in Angular would improve the interface, but it wouldn't stop someone from sending the request directly.

## 00:05:35.733 — What the current tests tell us

The current backend test run passed all twenty-one tests. They cover things like login, registration, trip responses, token checks, and password hashing. That's a useful starting point. It means I have checks I can run again while changing the code. But I need to be clear about what those tests actually do. Passing them doesn't mean I've tested every part of the app together.

For example, the route test on screen uses a test function that lets every request through. That's fine for checking whether a route calls the right handler. It doesn't tell me whether the real route blocks someone without permission. The controller tests also use fake database objects. The duplicate-code test supplies a database-style error; it doesn't actually create two records with the same code in MongoDB. And checking that the update includes runValidators doesn't prove that every field rule works in a real database.

I'd keep these fast tests and add a few tests that connect the real pieces. Those would use a separate test database and cover permissions, bad trip data, duplicate codes, and database failures. I'd also check the Angular form's loading and error states. I haven't run the live database or browser tests yet. When I report results later, I'll separate what was checked with fake objects from what was checked with the actual app and database.

## 00:06:52.333 — What the trip list does today

The current list function is pretty simple. It asks the database for all trips, then sends the whole list back. It doesn't filter by budget, compare trip lengths, or rank anything. That matters for the enhancement plan. I'm adding a way to rank trips; I'm not replacing a ranking system that's already here. This function shows where the trip data currently enters that part of the app.

Sending every trip is manageable with a small catalog. With a much larger catalog, there's more data to load, hold in memory, and send over the network. I can't tell exactly how long the database query will take just by reading this function. I'd need to measure it. But I can see that the response grows as the catalog grows. That gives me a reason to think about filtering and how many results the app really needs at once.

There is another behavior I'd review. An empty list currently returns a not-found response. The customer side treats that as an error. I'd rather have a normal empty list with a message like, no trips are available right now. A request for one specific trip that doesn't exist could still return not found. I'd update the tests and the client together if I make that change, so they agree on what an empty result means.

## 00:08:05.533 — Making prices and trip lengths usable

Before I rank trips, I need to fix how some of the values are stored. Price and trip length are both strings in the Angular type and in the database model. That works for showing text on a page. It gets awkward when I want to compare numbers. A label like three nights slash four days is easy for a person to read, but the ranking code shouldn't have to guess which number to use.

I'd store the price as a whole number of cents and the trip length as a whole number of days. For example, a price of one hundred fifty dollars and fifty cents would be fifteen thousand fifty cents. The page can still format that as a normal price. The same idea applies to trip length: store a clear number for calculations, then display a friendly label. Changing the TypeScript type alone won't fix the records already in MongoDB. Those records need to be checked and converted too.

I'd reject values that don't make sense instead of quietly treating them as zero. I'd also decide how rounding works and what the app should do with a zero budget. That's important if the score divides by the budget. Getting these rules settled first keeps the ranking code simpler. It can compare clean numbers instead of repeatedly trying to read numbers out of display text.

## 00:09:17.667 — Deciding what makes a trip a good match

The plan is to rank trips using a person's budget and preferred trip length. Before choosing an algorithm, I need to decide what a good match means. My starting idea is to leave out trips that cost more than the budget. Then, for the trips that remain, I'd give a better score to trips that fit the price and length preferences. That's a proposed rule. It isn't part of the current app yet.

For a simple first version, the price part of the score could reward money left in the budget. The length part could reward being close to the requested number of days. I'd start by giving those two parts equal weight. But that choice needs review. Someone might care much more about the length than getting the cheapest trip. For example, a very cheap two-day trip may be a poor match for someone asking for a week. I want the rule to reflect what the user actually wants, not just produce a number.

I'd also make ties predictable by using the trip code when scores are equal. The result count needs a limit too. In the algorithm discussion, I'll call that count k. If someone asks for five trips and only three qualify, return those three. Bad budgets, bad lengths, and invalid result counts should get a clear error. Both versions of the ranking code need to follow exactly the same rules.

## 00:10:30.467 — Starting with a simple ranking method

I'd build the straightforward version first. Give each matching trip a score, sort the trips from best to worst, and return the number requested. If the person asks for five, take the first five. This version is easy to read and gives me something to compare the more complicated version against. When both use the same score and tie rules, they should produce the same list.

There is a cost to sorting everything. If n is the number of matching trips, a comparison sort is usually described as big O of n log n. That describes how the work grows as the list gets larger; it isn't a stopwatch reading. This version also stores the scored list, so the extra space grows with the number of trips. For a small catalog, that may be completely reasonable. It may also be the simplest option when someone wants most of the available results anyway.

The code on screen is still the current list handler. I'd put the new ranking function in the trip service, rather than turn this controller into one long function. Getting trips from the database and ranking them are different jobs. I'd time them separately. Otherwise, a slow query could make the ranking method look slow even if most of the waiting happened before the ranking code started.

## 00:11:42.600 — Keeping only the best few trips

The other version would use a heap. A heap is a way of keeping a small collection organized so one important item is easy to find. Here, I'd keep only the best k trips seen so far, with the worst of those at the top. Suppose the user wants five results. Until I have five, I add each trip. After that, a new trip only gets in if it's better than the worst trip already kept.

That gives me a simple rule to check as the code runs: after each trip, the heap should still contain the best five seen so far. If the next trip is worse than all five, I can skip it. If it's better than the worst one, I swap them and fix the heap's order. Equal scores still need the same trip-code tie rule as the sorting version. Otherwise, the two versions could disagree even when the scores match.

For more than one requested result, selecting trips this way takes big O of n log k. Then I sort the final k trips for display. The heap only needs space for k items. If k is one, I can just keep the best trip seen so far. One limitation is that loading every trip first still uses memory for that whole list. A smaller heap doesn't remove that cost. I'd need to change how trips are read to get that larger memory saving.

## 00:12:53.867 — Checking the answers before checking speed

The first question is whether the ranking returns the right trips. I'd test the score and tie rules on their own, then compare the heap's result with the simple sorting version. I'd compare the full list, in order. Just checking that both return five items wouldn't catch a case where one of those five is the wrong trip.

I'd include an empty list, one matching trip, fewer matches than requested, and a request for just one result. I'd also try requesting all matching trips. Equal scores matter, so I'd include those too. Then I'd shuffle the input and check that the final result stays the same. That's why I want a clear tie rule. I'd also test bad numbers, invalid result counts, and large but valid prices. These cases should have agreed answers before I start measuring speed.

Once those checks pass, I'd try larger generated catalogs and different result counts. I'd run each test more than once, let the code warm up, and report the spread of the times instead of picking the nicest number. I'd measure ranking by itself and the full API request separately. I expect the heap to help when the user wants a few results from a large list. I haven't measured that yet. If the simpler sort is fast enough for this app, that's a useful result too.

## 00:14:06.700 — What the database already checks

Here is the Mongoose trip model. The fields are required, the code gets trimmed, and the schema asks for a unique code index. There is also an index declared for the name. So there are already checks and indexes in the design. My changes need to build on those rather than assume the database has no rules at all.

The gap is in what the values mean. Requiring a price string doesn't prove it's a valid price. In the earlier model check, a negative price string and a made-up trip length passed validation. That check ran against the model in memory; it didn't save those values to MongoDB. It still shows why I'd add number fields and sensible ranges. A field being filled in is different from it containing something the app can safely use.

I'd also check the indexes in the test database itself. Seeing unique in the schema tells me what the code asks for. It doesn't confirm which indexes are present in a running database. I haven't checked that live state yet. Before relying on the unique code rule, I'd confirm the index exists and test an actual duplicate insert. For the name index, I'd check whether the app's queries use it. I don't want to add or keep indexes just because they sound useful without checking what they do.

## 00:15:18.367 — Checking input and updating old records

This helper copies only the trip fields the app allows. That's useful because a request could contain extra properties that shouldn't be saved. The next helper checks for missing values, null values, and empty strings. I'd keep those checks. But a negative price can still be filled in, and a made-up length can still be a nonempty string. So I also need checks that ask whether the values make sense.

I'd make the server, database model, and form agree on the number fields and allowed ranges. Then I'd check the existing records before changing them. I'd make a backup, run the conversion without saving first, and report anything that couldn't be converted safely. After the actual update, I'd check the record counts and converted values. I'd also test both adding and editing a trip, because a rule that only works on new records leaves a gap when old records are changed.

The error helper needs a small cleanup too. It currently sends the database validation message back in a details field. That can include internal names or rejected values. I'd give the user a clear message about the field they need to fix, while keeping the full technical error in controlled server logs. The goal is useful feedback without sending every database detail to the browser. I'd test both ordinary input mistakes and unexpected database failures.

## 00:16:30.467 — Checking whether an index actually helps

This lookup asks for one trip by its code. A code index is a natural fit for that. The list query is different: right now it asks for everything. Once I add search rules, the query might filter by budget, dates, or trip length. I'd choose any new indexes based on those actual queries. Adding an index to every field would create more work without showing that the app is faster.

To check an index, I'd use a separate test database with enough sample trips to make the comparison useful. I'd run the query and use MongoDB's explain output to see how it found the results. I'd record how many results came back, how many records and index entries it checked, and how long it took. Then I'd add the proposed index and repeat the same test. I'd try both a narrow search with a few matches and a broad search with lots of matches.

Indexes also take up space and need updating when records change. That's part of the decision. An index may help find trips under a budget, but it won't automatically speed up a preference score calculated later in JavaScript. I'd keep those measurements separate so I can tell where any improvement came from. This part of the project is about learning which database changes help this app, and being able to show the results instead of guessing.

## 00:17:45.333 — Keeping database setup and test data safe

The database connection can be set through environment variables, and there is code for closing it when the app shuts down. Those are useful features. One thing I'd change is this connection message. It prints the whole connection string. The local default doesn't include a password, but a different connection string could. I'd print a simple connection message or just the safe parts, such as the database name. I didn't find a leaked password here; the concern is what this log statement could print with a different setup.

The seed script is another thing I'd handle carefully. It deletes the trips and then loads the sample data. That's fine for a throwaway development database, but it's not something I'd run against records I need to keep. I haven't used it to wipe the working database. For the new tests, I'd use a clearly separate database and make sure the setup refuses to run against the wrong one. Cleanup should only remove data created for those tests.

I'd also test what happens when the database isn't available and when the app closes. The README should explain how to run these checks and which database they use. That makes the work easier for someone else to repeat. I'd keep the original project and the record-conversion notes too, so there is a clear starting point to go back to if a change causes problems.

## 00:19:01.600 — The main issues I'd work on first

Looking back over the review checklist, the app already has a useful starting structure: separate routes, models, and two clients. The main design change is to split the trip controller into smaller jobs. The README and comments also give me a starting point. I'd update them to explain permissions, price units, ranking rules, and how to run the tests. I don't need a comment on every line; I need explanations where someone might otherwise have to guess.

For values and calculations, prices and trip lengths are the big issue. For loops and decisions, I'd focus on making the heap easy to check, including ties and empty inputs. There is also a simple array filter in the Angular list that removes a deleted trip from the screen. That seems reasonable for this job. I wouldn't replace every simple loop just to use a more advanced data structure. The biggest protection gaps are admin permissions, checks on actual values, and what the app reveals in errors and logs.

JavaScript handles memory cleanup differently from languages where I would free memory myself. Here, I'd watch how much trip data gets loaded, how large the responses get, and whether connections are closed properly. My order of work would be to settle the rules and permissions first, keep the existing tests passing, and then add ranking and database improvements. It's easier to judge performance once I know the app is doing the right thing.

## 00:20:23.567 — How this fits the capstone plan

This work also gives me a way to show the five course outcomes. For working with others, I'd use instructor feedback and write down the decisions that still need review. For communication, I'd explain the changes in the README and project write-ups using examples like the ones in this walkthrough. Someone reviewing the project should be able to understand what changed and why, without having to work it all out from the code.

For algorithms, I'd compare sorting with the heap, check that they return the same answers, and explain when each one makes sense. For tools and development practices, I'd use tests, source control, and the database query results. For security, I'd work on admin permissions, input checks, error messages, and safe database setup. Those are the things I plan to demonstrate. I'll report them as completed once the changes and test results are actually there.

The planned order is software design in Module Three, algorithms and data structures in Module Four, and databases in Module Five. I'll need to agree on prices and trip lengths early because both ranking and the database changes depend on them. I'll keep the original project, make focused changes, and use feedback before moving on. That covers how the app works now, the problems I'd address, and how I'd check the changes. Thanks for taking a look.
