# Redis SCAN request handler

## Core imports

	url = require 'url'


## Library imports

	metah = require 'metah'

	R = require 'ramda'

	{always, append, applySpec, curry, equals, flatten, flip, ifElse, join, juxt, nth, pipe, prop, tap} = R


## Relative imports

	get_url_from_request = require './get_url_from_request'

	scan_html_maker = require './scan_body'

	send_error = require './send_error'

	send_ok = require './send_ok'


## Parse URL to Redis command

	get_count_from_url_query = pipe(prop('count'), flip(R.or)(12))

	get_cursor_from_url_query = pipe(prop('cursor'), flip(R.or)(0), parseInt)

	# add_pattern = ifElse(equals(''), always([]), flip(append)(['MATCH']))
	get_pattern_from_url_query = pipe(prop('pattern'), flip(R.or)('*'))

	get_command_from_request = pipe(get_url_from_request.query, juxt([
		get_cursor_from_url_query,
		pipe(get_pattern_from_url_query, flip(append)(['MATCH'])),
		pipe(get_count_from_url_query, flip(append)(['COUNT']))
	]), flatten)


## Redis SCAN command

	redis_scan_from_request = (log, client, request)->
		new Promise (resolve, reject)->
			command = get_command_from_request(request)

			log.trace("Redis command is `SCAN #{join(' ', command)}`.")

			client.scan command, (err, result)->
				if err
					log.error(err: err, 'Redis SCAN failed.')

					reject err

				else
					log.trace
						redis_scan_cursor: result[0]
						redis_scan_values: result[1]
					, 'Redis SCAN completed.'

					resolve result


## Exports

### HTML

	module.exports.html = curry (h, log, redis_client, request, response)->
		log.trace req: request, 'Started scan.html handler.'

		fix_scan_result = applySpec
			keys: nth 1
			next_cursor: nth 0
			query: always get_url_from_request.query request

		make_html = scan_html_maker h

		# scan_is_done = tap log_trace 'Redis SCAN done; building page.'

		make_page_from_scan_result = pipe(fix_scan_result, make_html, send_ok.html(response))

		redis_scan_from_request log, redis_client, request
		.catch send_error.html_500('Redis SCAN failed.', log, request, response)
		.then make_page_from_scan_result


### JSON

	module.exports.json = curry (log, redis_client, request, response)->
		log.trace req: request, 'Started scan.json handler.'

		redis_scan_from_request log, redis_client, request
		.catch send_error.json_500('Redis SCAN failed.', log, request, response)
		.then send_ok.json(response)
