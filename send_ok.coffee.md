# Send OK

## Library imports

	{curry} = require 'ramda'


## Exports

	module.exports.css = curry (response, content)->
		response.writeHead 200, 'Content-Type': 'text/css'
		response.end content


	module.exports.html = curry (response, content)->
		response.writeHead 200, 'Content-Type': 'text/html'
		response.end content


	# module.exports.js = curry (response, content)->
	# 	response.writeHead 200, 'Content-Type': 'text/javascript'
	# 	response.end content


	module.exports.json = curry (response, content)->
		response.writeHead 200, 'Content-Type': 'application/json'
		response.end JSON.stringify content
