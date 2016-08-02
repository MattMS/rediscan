# Get URL from request

## Core imports

	url = require 'url'


## Library imports

	{flip, pipe, prop} = require 'ramda'


## Get URL from request

	get_url_from_request = pipe(prop('url'), flip(url.parse)(true))


## Exports

	module.exports.path = pipe(get_url_from_request, prop('pathname'))

	module.exports.query = pipe(get_url_from_request, prop('query'))
