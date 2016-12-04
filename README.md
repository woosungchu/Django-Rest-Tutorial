##1.Serialization
<http://www.django-rest-framework.org/tutorial/1-serialization/>

####. Model -> Serializer -> JSONRenderer
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
##3.Class based views
##4.Authentication and permissions
##5.Relationships and hyperlinked APIs
##6.Viewsets and routers
##7.Schemas and client libraries
