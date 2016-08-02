# Send error

## Library imports

	{curry} = require 'ramda'


## 404

	module.exports.text_404 = curry (request, response)->
		response.writeHead 404, 'Content-Type': 'text/plain'
		response.end 'Invalid path requested.'


## 500

	module.exports.html_500 = (message, log, request, response)->
		(err)->
			log.error message, err: err

			response.writeHead 500, 'Content-Type': 'text/plain'
			response.end err


	module.exports.json_500 = (message, log, request, response)->
		(err)->
			log.error message, err: err

			response.writeHead 500, 'Content-Type': 'application/json'
			response.end JSON.stringify
				error: true
				message: message
