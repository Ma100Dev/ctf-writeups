# Rock-Paper-Scissors

## Basic details
The application code can be viewed [here](https://static.cor.team/corctf-2024/0ed9a91a06fb2ef38b447134419c109d015cdf8a1ace4bb157b92d739f8a2c9e/rock-paper-scissors.tar.gz).

The application in question is a Node JS based web app that allows the user to play rock-paper-scissors against a computer that randomly selects a move to play after choosing a username.

As seen below in the shortened code block, the  `/flag` endpoint will reveal the flag to the user, but only after their score exceeds 1336 points.
```
app.get('/flag', async (req, res) => {
	// Authentication logic here... 
	const score = await redis.zscore('scoreboard', req.user.username);
	if (score && score > 1336) {
		return res.send(process.env.FLAG || 'corctf{test_flag}');
	}
	return res.send('You gotta beat Fizz!');
})
```
The application requires one to register an account before playing. Any username can be chosen regardless of whether it is already used. This would allow anyone to play as anyone else! One user account already exists, but it sadly only has 1336 points. Since this is not enough a slightly more involved exploit will be necessary. 

## The exploit
At the new game endpoint `/new` the username is assigned as follows:

`const { username } = req.body;`

The type of the 'username' is not ever checked! It will get saved to the signed JWT as is. Maybe we can send something other than a string, such as an object, to exploit the app and get the required score?

In the game logic, the following code can be found:
```
// If the player defeats the computer, continue
if (winning.get(system) === position) {
	const score = await redis.incr(game);
	return res.send({ system, score, state: 'win' });
// Else delete the game and save its score.
} else {
	const score = await redis.getdel(game);
	if (score === null) {
		return res.status(404).send({ error: 'game not found' });
	}
    // Whatever is assigned to out username gets sent here. Interesting!
	await redis.zadd('scoreboard', score, username); 
	return res.send({ system, score, state: 'end' });
}
```
If the input is a string as implicitly expected, this will work just fine. But many values can be set at once! If we send an array as JSON as the username (`["some_guy", 9999, "joe"]`), instead of the data being set to roughly `['some_guy': score_here]`, the result will instead be something like `['some_guy': score_here, 'joe': 9000]`. After saving our score as such a user with such a specially crafted 'username', the flag will become available at the `/flag` endpoint. $\square$

<sup>(full exploit at [exploit.py](exploit.py))</sup>