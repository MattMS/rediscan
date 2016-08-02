# Redis browser HTTP server

## Core imports

	# fs = require 'fs'

	url = require 'url'


## Library imports

	bunyan = require 'bunyan'

	h = require 'hyperscript'

	{always, append, apply, bind, curry, drop, equals, flatten, flip, ifElse, join, juxt, pipe, prepend, prop, slice, T, tap} = require 'ramda'


## Relative imports

	base_page_maker = require('./base_page')(h)

	get_url_from_request = require './get_url_from_request'

	handle_scan = require('./scan')

	send_error = require('./send_error')

	send_ok = require('./send_ok')


## Logging

	log = bunyan.createLogger
		level: 'trace'
		name: 'server'
		serializers: bunyan.stdSerializers
		# serializers:
			# err: bunyan.stdSerializers.err
			# req: bunyan.stdSerializers.req
			# res: bunyan.stdSerializers.res
		# serializers:
		# 	chunk: pick(['field'])
		# stream: fs.createWriteStream('server.log.json')

	# log_error = bind(log.error, log)
	# log_info = bind(log.info, log)
	log_trace = bind(log.trace, log)


## Redis GET handler

	wrap_code = pipe(flip(append)(['pre']), apply(h))


	handle_get_html = curry (redis_client, request, response)->
		log.trace 'Started /get/ handler.', req: request

		key = pipe(get_url_from_request.path, drop(5))(request)

		send_result = pipe(
			# NOTE: This fails because it receives `result` as a parameter.
			# tap(log_trace('Redis GET done; building page.')),
			wrap_code,
			base_page_maker,
			send_ok.html(response)
		)

		redis_get redis_client, key
		.catch send_error.html_500('Redis GET failed.', log, request, response)
		.then send_result


	redis_get = (client, key)->
		new Promise (resolve, reject)->
			client.get key, (err, result)->
				if err
					reject err
				else
					resolve result


## Exports

	handle_scan_html = handle_scan.html(h)

	handle_scan_json = handle_scan.json(h)

	module.exports = (redis_client)->
		(request, response)->
			log.trace 'Received request.', req: request

			# pipe(get_url_from_request.path, cond([
			# 	[propEq('pathname', '/'), handle_scan_html(redis_client)],
			# 	[pipe(prop('pathname'), slice(0, 5), equals('/get/')), handle_get_html(redis_client)],
			# 	[propEq('pathname', '/scan.json'), handle_scan_json(redis_client)],
			# 	[T, handle_missing_page]
			# ]))(request, response)

			url_path = get_url_from_request.path(request)

			if '/' == url_path
				handle_scan_html redis_client, request, response

			# else if '/index.js' == pathname
			# 	handle_ok_js request, response

			else if '/get/' == url_path.slice 0, 5
				handle_get_html redis_client, request, response

			else if '/scan.json' == url_path
				handle_scan_json redis_client, request, response

			else
				send_error.text_404 request, response
