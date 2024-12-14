# Hierarchy Optimisation

# TL;DR

Admin panel performance suffers due to fetching the full hierarchy to compute resource names.

Options considered for solving this

- Create a new endpoint that returns the parent of the resource with the id and full_name
- Create a new endpoint that builds only the section of the tree we're interested in
- Modify the exisiting fetch to only compute data involved in the current level (We chose this one)

# Background/Problem

Our content in our database is set up so that it's a hierarchy. We use this hierarchy to calculate the full name of a child lower down in the tree. For example, we have the following hierarchy.

curriculum -> subjectGroup -> subject -> topic, the full name of the topic has this structure:

```ts
`${curriculum.short_name} ${subjectGroup.short_name} ${subject.short_name} - ${topic.name}`;
```

So an example of this would look like: `IB Math SL - Differentiation`.

Currently, in our admin panel, in order to generate these full names, we are having to make a request for the entire hierarchy which takes about ~10-15 seconds to load. We are currently caching the hierarchy, but it has a stale time (how long until the data is considered old, and needs to be refetched) of 60 seconds, so we can really only utilise the cache for a minute before we need to make another 10-15 second request for a new hierarchy. This means that for some resources, we are fetching a lot of unnecessary data that isn't going to be used, but is still required to be fetched.

An observation with the admin panel is that we only ever need the parents of the resource up to the root node. This means we can optimise how much data we are requesting as a hierarchy. Instead of requesting the entire hierarchy, we can be selective with how much of the hierarchy we need per request.

In theory, the most optimal we can be is ask for a specific level in the hierarchy, and only generate the tree for the children above the current level. As we are generating the tree, we _could_ also add the full name and parent name for a specific resource in the return body to prevent the use of extra tree traversal and name generation in the frontend. However, the frontend traversal is only ever going to be as deep as the number of nodes in the branch, so this is inexpensive as a frontend computation.

# Database Hierarchy

Here is an overview of the hierarchy in the database.

![[graph (1).png]]

# Potential Solutions

This section looks at some potential solutions for solving this problem. The three main options I considered were:

1. Create a new endpoint which takes a level in the tree and return only the name information of the parent.
2. Create a new endpoint that builds the only the tree from the specified level upwards
3. Update the existing hierarchy endpoint to take a level and use it to select nodes in the tree.

## Ranking characteristics of each option

| Characteristics       | Options |     |                |
| --------------------- | ------- | --- | -------------- |
|                       | 1       | 2   | 3              |
| Return Body Size      | 1st     | 2nd | 3rd (marginal) |
| Fastest to complete   | 3rd     | 2nd | 1st            |
| Fastest Response      | 1st     | 2nd | 3rd (marginal) |
| Least change required | 3rd     | 2nd | 1st            |
| Least Duplication     | 2nd     | 3rd | 1st            |

### Scores

Not the best metric to decide, but this was considered

1. 1 + 3 + 1 + 3 + 2 = 10
2. 2 + 2 + 2 + 2 + 3 = 11
3. 3 + 1 + 3 + 1 + 1 = 9

Summing the scores, option 3 was marginally better for the considered characteristics.

1. Create a new endpoint which takes a level in the tree and return only the name information of the parent

This involves creating a new endpoint that will take in the level of the hierarchy we are interested in, and return only the parent with the full_name populated. Within the admin panel, we can use the `parent_id` for a specific resource is to traverse up the tree get the resource's full name. This is because the only fields we are interested in within the hierarchy is the name of the parents for the specified resource. This would return a list of objects that looks like:

```json
[
  { id, full_name },
  { ... }
]
```

Then we can just find the parent from the current resource and use the foreign key or many-many table to fetch the correct name.

**Benefits**

- Return body is small
  - We only care about the id and full_name of the parent.
- Time efficient as we can filter and populate as we fetch the resource

**Drawbacks**

- Each resource will have a different query and require a complex setup to creating the query per resource.
- Will be taking a lot of code from the hierarchy, meaning two different methods of traversal.

---

2. Create a new endpoint that builds the only the tree from the specified level upwards

We create a new endpoint that will build a tree that goes from the root to the specified level. The idea here is that if you know the child, you know what the parent is, and you could theoretically build your way up until we get to the root node (in this case `curriculum`). This way you don't need to have any knowledge of the other branches in the tree.

**Benefits**

- Can reuse the types from the existing hierarchy with the exceptions of using `.pick` and `.omit` from lodash
- Each request will be smaller than the full hierarchy request.

**Drawbacks**

- Rewriting this will involve a bit of duplication from the original hierarchy builder.
- Writing the logic for calculating the types and parents for a specified level will add a lot of extra complexity.

3. Update the existing hierarchy endpoint to take a level and use it to select nodes in the tree.

We can add a new input to the current hierarchy that will take in a level, and we can selectively allow specific queries to run depending on the level that we are currently interested in. The logic for this will check which branch we are in and pass through all the valid nodes in the branch that would make up the tree.

#### Benefits

- Doesn't require rewriting new endpoints
- Faster for queries that utilise the level
- Don't need to update the return type as it'll just have empty entries.

#### Drawbacks

- Adds more to the current endpoint
- Will need to update the frontend caching logic as this won't work anymore.

# Chosen Solution

After evaluating the options, **Option 3** was selected: Update the existing hierarchy endpoint to take a level and use it to select nodes in the tree. This approach balances efficiency and simplicity while avoiding the need to rewrite or duplicate existing code.

## Why Option 3?

- **Minimal Code Changes**: Leveraging the current hierarchy logic reduces the risk of introducing new bugs or duplicating traversal code.
- **Performance Gains**: By limiting queries to the relevant branch, we significantly reduce the load on both the database and the API.
- **Frontend Compatibility**: Since the frontend only requires lightweight traversal to compute the full name, no additional modifications are necessary.

## Implementation Details

1. **Filter Hierarchy by Level**:
   - Introduced a new input parameter, `level`, to specify the depth of the hierarchy being fetched.
   - A helper function, `getAllowedLevels(level)`, determines which levels are relevant for the request. For example:

```ts
getAllowedLevels("caseStudyGroup");
// Returns: ['curriculum', 'subject_group', 'subject', 'case_study_super_group', 'case_study_group']
```

2. **Modify Queries Dynamically**:
   - Each `unionAll` query now checks if the level is within the allowed levels before being executed. For example:

```ts
const queries = [];
if (isInBranch(allowedLevels, 'curriculum')) {
  queries.push(client('curriculum').select(...));
}

if (isInBranch(allowedLevels, 'subject_group')) {
  queries.push(client('subject_group').select(...));
}

const rows = await client.unionAll(queries));
```

3. **Default Behaviour**:
   - If no `level` is specified, the endpoint defaults to fetching all levels, preserving current functionality.

## Benefits

- **Improved Efficiency**: Requests are smaller and more targeted, reducing load times significantly.
- **Backward Compatibility**: Existing features continue to work seamlessly, ensuring no breaking changes if the level is not specified.
- **Frontend Simplicity**: Full names are still generated quickly on the frontend by traversing up the tree, avoiding unnecessary backend computation.

## Performance Impact

Testing with large datasets showed remarkable improvements:

- For a 164 MB database, fetching a full hierarchy took **4 seconds**, while fetching by level (where `depth=4`) reduced this to **35 milliseconds**.

| DB Size | Full tree | d = 4 | d = 5 | d = 6 | d = 7 | d = 8 |
| ------- | --------- | ----- | ----- | ----- | ----- | ----- |
| 55 MB   | 280ms     | 50ms  | 60ms  | 75ms  | 70ms  | 102ms |
| 164 MB  | 4s        | 35ms  | 40ms  | 85ms  | 360ms | 2.3s  |

_note 1: d = length of nodes in the branch or depth of the hierarchy._
_note 2: The data is more concentrated in two branches, and the rest of the data is relatively small for the other shorter branches._

---

By implementing this solution, we achieved significant performance gains with minimal disruption to existing functionality. The focused design ensures scalability as the hierarchy grows, while retaining flexibility for future enhancements.

# Limitations

## Endpoint only takes one level

The endpoint currently only allows you to specify one level. Future work could involve being able to pass multiple levels into the endpoint to allow different branches of the tree to built in the same request. In our use case, this is only applicable for hierarchies that involve the `paper` resource. However, this adjustment can be made as our hierarchy usage becomes more complex, perhaps.

# What have I learnt

## Take a step back

In the past, I have had a habit of being assigned a task, and then going ahead and trying to solve it straight away without much proper thought. What this lesson means to me, is that when trying to solve a problem, it is better to take the time in planning, investigating the problem and potential solutions before going ahead and trying to get straight into the code.

From taking this approach, I was not only able to understand what the problem was, but also by looking at possible solutions, and chatting through those with colleagues, helped to choose the better solution before even touching the code.

My initial solution was option 1 above, and if I hadn't taken the time to think about other solutions, I probably would've just tried to implement that. Then, I would get to the review stage, and then probably have some changes requested, and need to take some time to rethink the solution.

## Analysing a system

Throughout doing this project, it helped me understand how to analyse a system better. What does that mean? To me, that means being able to look at how something is implemented, and think about the 'whys'. Why was this done the way it was. What is the reason why this current process is undesired or slow? What changes can I make that is going to solve the problem that I am currently facing.

For this problem, I was able to understand what we were doing with the hierarchy, and then realising what optimisations I was able to make to cut the loading times down by at least half in the worst case (when using the level).

## Personal Growth

Just from this project alone, as well as being intentional in the processes I am executing. I was able to put to practice some of the skills that I have been trying to work on to help boost my career development, the main skills that I felt that this was able to highlight were:

- Documentation and Knowledge Sharing
  - This blog is a good example of this. Prior to writing this blog, I had created a design document. This is reflected in the background, problem, and potential solutions sections in this blog.
- Project Ownership
  - Taking on a project and being confident in the direction the project is going.
- Noticing a problem, and coming up with a solution
  - Before making these changes, I had heard from content creators in the company that this admin panel was a pain point for creating content due to how slow it was. Given this knowledge, I was able to see the issue, and then come up with a solution that will help to alleviate some of the stress from our content creators.

Thankful for my team and manager to allow me to take the time to make these changes.
