# SCAN page script

	# module.exports = ({document, window, XMLHttpRequest})->
	module.exports = ->

- https://developer.mozilla.org/en-US/docs/Web/API/Node/lastChild

		check_for_call = (cursor, ul)->
			li = ul.lastChild

			if 0 != cursor and li and li.getBoundingClientRect().bottom < window.innerHeight
				call_scan_json cursor, handle_scan_json


## Perform JSON request

		call_scan_json = (cursor, callback)->
			request = new XMLHttpRequest()

			request.onreadystatechange = (event)->
				req = event.target

				if XMLHttpRequest.DONE == req.readyState
					if 200 != req.status
						console.error(req)

					else
						handle_scan_json JSON.parse req.responseText

			search_input = document.getElementById 'search_input'

			pattern = search_input.value

The `true` parameter indicates that this request is asynchronous.

- https://developer.mozilla.org/en-US/docs/AJAX/Getting_Started

			request.open('GET', "/scan.json?count=12&cursor=#{cursor}&pattern=#{pattern}", true)
			request.send()


## Handle JSON response

		handle_scan_json = (json_response)->
			[cursor, values] = json_response

			ul = document.getElementById 'search_results'

			for key in values
				li = document.createElement 'li'

				a = document.createElement 'a'
				a.setAttribute 'href', "/get/#{key}"
				a.innerHTML = key

				li.appendChild a
				ul.appendChild li

			ul.setAttribute 'data-last-cursor', cursor

			check_for_call cursor, ul


## Add event listeners

		check_for_call_from_event = ->
			ul = document.getElementById 'search_results'

			cursor = ul.getAttribute 'data-last-cursor'

			check_for_call cursor, ul

		document.addEventListener 'DOMContentLoaded', (event)->
			search_nav = document.getElementById 'search_nav'

- https://developer.mozilla.org/en-US/docs/Web/API/HTMLElement/style

			search_nav.style.display = 'none'

			check_for_call_from_event()

- https://developer.mozilla.org/en-US/docs/Web/Events/scroll

		window.addEventListener 'scroll', (event)->
			check_for_call_from_event()
