---
layout: post
title:  "Create a neuron for Kalliope - Allocine"
date:   2019-01-22 17:53:02 +0100
categories: [kalliope]
---
On may way to discover and play with [Kalliope](https://kalliope-project.github.io), I decided to build a neuron which I can
then open-source and expose to the community.

I initially wanted to build a neuron to
fill my shopping cart on my nearest Supermarket. After some research, it appears there
is no public API nor an API source that someone would have build after doing some
reverse engineering. That does not mean it is not possible but I would have spend more
time working and discovering APIs or simulating the POST operation that we can do on the
website than really building a neuron.

I decided to build a neuron to integrate Kalliope with Allocine which is a french
website to gather information about movies, theaters, actors, etc. As for the
supermarket, there is no public API but multiple [custom API](https://github.com/thomasRousseau/python-allocine-api) have been created by
reverse engineering the mobile apps. My goal would be
to build a neuron so people can build their own brains to get the list of movies in
their favorite theaters, the show times on a given date so we don't have to open
a browser to get these information.

This post does not aim to detail how the neuron works and how it has been built but
only to say it is here, given I have not requested it to be added to the [Kalliope
contributed neurons list](https://kalliope-project.github.io/neurons_marketplace.html) for now. This is still a work-in-progress and is very basic for now.
It only allow to create a command which would list the movies and their showtime 
in a preselected theater for the current day.

I will try to first improve this feature so at least we can ask for another day
than today. Next plan is to have the possibility to get only the list of movies 
(not associated with showtimes) and to get the showtime for a specific movie only.

If you have feedback or if you wish to help, [issues](https://github.com/vbouchet31/kalliope-allocine/issues) and 
[pull requests](https://github.com/vbouchet31/kalliope-allocine/pulls) on [github
repository](https://github.com/vbouchet31/kalliope-allocine) are welcomed.


Source: [Kalliope Allocine neuron](https://github.com/vbouchet31/kalliope-allocine)