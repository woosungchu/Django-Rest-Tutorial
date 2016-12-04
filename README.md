##1.Serialization
<http://www.django-rest-framework.org/tutorial/1-serialization/>

#### Model -> Serializer -> JSONRenderer
- snippet = Snippet(code='print hello, world')
- snippet.save()
- serializer = SnippetSerializer(snippet)
- content = JSONRenderer().render(serializer.data)

#### Stream -> JSONParser -> Serializers -> Model
- stream = BytesIO(content)
- data = JSONParser().parse(stream)
- serializer = SnippetSerializer(data=data)
- serializer.is_valid()
- serializer.save()

##2.Requests and responses

JSONResponse(HttpResponse) ->  rest_framework.response.Response <br/>
@csrf_exempt -> @api_view(['GET','POST']) <br/>

> @api_view(['GET', 'PUT', 'DELETE']) <br/>
> def snippet_detail(request, pk): <br/>
>     try: <br/>
>         snippet = Snippet.objects.get(pk=pk) <br/>
>     except Snippet.DoesNotExist: <br/>
>         return Response(status=status.HTTP_404_NOT_FOUND)
>
>     if request.method == 'GET':
>         serializer = SnippetSerializer(snippet)
>         return Response(serializer.data)
>
>     elif request.method == 'PUT':
>         serializer = SnippetSerializer(snippet, data=request.data)
>         if serializer.is_valid():
>             serializer.save()
>             return Response(serializer.data)
>         return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)
>
>     elif request.method == 'DELETE':
>         snippet.delete()
>         return Response(status=status.HTTP_204_NO_CONTENT)

Httpie test request
- http http://127.0.0.1:8000/snippets/ Accept:application/json  # Request JSON
- http http://127.0.0.1:8000/snippets/ Accept:text/html         # Request HTML
- http http://127.0.0.1:8000/snippets.json  # JSON suffix
- http http://127.0.0.1:8000/snippets.api   # Browsable API suffix
- http --form POST http://127.0.0.1:8000/snippets/ code="print(123)"
- http --json POST http://127.0.0.1:8000/snippets/ code="print(456)"

##3.Class based views
##4.Authentication and permissions
##5.Relationships and hyperlinked APIs
##6.Viewsets and routers
##7.Schemas and client libraries
