---
layout: post
title: Consistent Hashing
type: post
---

Suppose we want to provide a percentage based rollout of features to a set of users without storing the full set of features for each user.  It may sound overkill, but if you have millions of users and 50 features, suddenly you are talking about storing 50M items and thats no small amount.

If the user id's are evenly distributed in a mathematical space (say they're a UUID), you can leverage this to project the distributed ID onto a number line, scale the number to between 0 and 1, and then use that as a flag for if the user gets the feature X or not.

Lets look at an example:

```js
import { createHash } from 'crypto';

/**
 * Hashes the value consistently to a space between 0 and 1. Can be used for determining if certain
 * id's have feature flags enabled automatically (for probabilistic rollouts, etc)
 * @param value The value to convert to a number and convert to a 0 to 1 representation on the number line
 * @param offset The offset on the number line to put the hashed value
 * @param scale Default scale is the entire space of md5 numbers
 */
export function consistentHash(value: string, offset = 0, scale = 1): number {
  const buffer = createHash('md5')
    .update(value)
    .digest();

  const hashNum = buffer.readUInt32BE(0);
  
  const maxMd5 = Math.pow(2, 32)
  
  const numberLineMax = maxMd5 * scale

  // scale the hash by the total number space
  return ((hashNum + offset) % numberLineMax) / numberLineMax;
}
```

If our user id `6f805e32-592e-46a2-95f3-51826f27e74f` we can hash it to a projected value and

```js
consistentHash('6f805e32-592e-46a2-95f3-51826f27e74f')
```

Gives us: `0.8326200970914215`

So user `6f805e32-592e-46a2-95f3-51826f27e74f` is _always_ going to get a feature if its configured to be 83% likely.  That's kind of cool cause you don't need to store the result of this anymore, you can just _know_.  

Buut, now we have a new problem.  What happens to poor ol user X who always hashes to `0.01`.  They get every experiment you throw at them because they're constantly in the 1% group.

This is where the other paramters of our method come into play like `offset`.  Imagine that we've projected the user onto the number line:

```
0----X-------------2^32
```

Our 2^32 comes from the fact that we are using md5 to hash the value and the max md5 value is 2^32, which would be the end of our number line. 

We could _offset_ our projected value and then loop it around the number line, to create a consistent but moved percentage

```
X+N

0----(X)-----------(X+N)--2^32
```

This now gives us the same distribution but allows our poor user to not always be in the 1% group.  How do we choose the offset? Well we can hash any string to get a number, so we can use the experiment/feature ID to do that. Imagine your feature is `"uses_new_login_flow"` you can hash that to a number N and apply that to the number line.

## Variants

Now that we can hash users to percentages, what if we could then do experiments with variants. For example, imagine we want to see what happens to 20% of our users if we give them one of 3 choices, where each choice is equally likley. Call the choices variants 1, 2, and a control.  This happens all the time that we'd want to experiment on behavior and see what the outcomes are.

Putting a user into an experiment is easy, we just did that above.  But how do we put someone into equal probability variants? If a user falls into 33% they'll fall into the other 33% as well.  

Lets walk through an example and see where it leads us. 

Going back to our user`6f805e32-592e-46a2-95f3-51826f27e74f`  who we know maps to `0.83`.  Lets say our experiment is to 50% of our users.  Cool, we know this user is going to be in the experiment.

We now want to test to see if this user gets variant 1, 2 or our control (variant 3).  What we want to do is deterministically dice roll this user to fall into 1 of 3 buckets consistently.

We know that the variants are equally probably, so we have a number line like this:

```
 1   2   control
0---|---|---?
```
What we don't know is what the end of the number line should be and how to put someone on it.

What if we re-hashed the user id, but _scaled_ it so that instead of the max number being the max of md5, that we scale it to 

```
percentage_experiment * max_md5
```

ie.

```
0.5 * 2^32
```

We know the user falls into the bucket of 50%, so if we constrain the line to now be 50% of the max we can see where they now fall on _that_ line and then test.

If we revisit the method we can now do

```js
consistentHash('6f805e32-592e-46a2-95f3-51826f27e74f', 0, 0.5)
```

which now gives us `0.665240194182843`

This tells us where on the number line this user would fit GIVEN they were already selected for the past experiment.

If we wanted to calculate 3 equally likley probabilities, we can divide the number line into probabilities and say 

> The first is from 0 to 0.33. 
> The second is from 0.33 to 0.66
> The third is from 0.66 to 1

So we can now see that `6f805e32-592e-46a2-95f3-51826f27e74f` would fall into our second variant.

Doing this calculation is 100% reproducible, is O(1), and doesn't require any external storage costs! 

## Conclusion

Doing consistent hashing (mapping distributed values to a number line) has a lot of fantastic uses, but does require that you have evenly distributed data to start with.  If you have evenly distributed values leveraging consistent hashing can save storage and IO costs quite a bit, at least until you have something more complicated :p
 

