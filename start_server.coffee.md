# Start server

## Core imports

	http = require 'http'


## Library imports

	bunyan = require 'bunyan'

	h = require 'hyperscript'

	redis = require 'redis'


## Relative imports

	request_handler_maker = require './server'


## Logging

	log = bunyan.createLogger
		level: 'trace'
		name: 'start_server'
		# stream: fs.createWriteStream('start_server.log.json')

	# log_error = bind(log.error, log)
	# log_info = bind(log.info, log)


## Redis client

	redis_client = redis.createClient()

	redis_client.on 'error', (err)->
		log.error 'Redis client threw an error.', err: err


## Run!

	listener = request_handler_maker redis_client

	server = http.createServer listener

	hostname = 'localhost'

	port = 1337

	server.listen port, hostname, ->
		log.info "Started server listening on #{hostname}:#{port}."

	# redis_client.quit()
