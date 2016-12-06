##1.Serialization
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

    @api_view(['GET', 'PUT', 'DELETE'])
    def snippet_detail(request, pk):

       try:
           snippet = Snippet.objects.get(pk=pk)
       except Snippet.DoesNotExist:
           return Response(status=status.HTTP_404_NOT_FOUND)

       if request.method == 'GET':
           serializer = SnippetSerializer(snippet)
           return Response(serializer.data)

       elif request.method == 'PUT':
           serializer = SnippetSerializer(snippet, data=request.data)
           if serializer.is_valid():
               serializer.save()
               return Response(serializer.data)
           return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)

       elif request.method == 'DELETE':
           snippet.delete()
           return Response(status=status.HTTP_204_NO_CONTENT)


####Httpie test request
- http http://127.0.0.1:8000/snippets/ Accept:application/json  # Request JSON
- http http://127.0.0.1:8000/snippets/ Accept:text/html         # Request HTML
- http http://127.0.0.1:8000/snippets.json  # JSON suffix
- http http://127.0.0.1:8000/snippets.api   # Browsable API suffix
- http --form POST http://127.0.0.1:8000/snippets/ code="print(123)"
- http --json POST http://127.0.0.1:8000/snippets/ code="print(456)"

##3.Class based views
####From @api_view to APIView

    from snippets.models import Snippet
    from snippets.serializers import SnippetSerializer
    from django.http import Http404
    from rest_framework.views import APIView
    from rest_framework.response import Response
    from rest_framework import status

    class SnippetList(APIView):
        def get(self, request,format=None):
            snippets = Snippet.objects.all()
            serializer = SnippetSerializer(snippets, many=True)
            return Response(serializer.data)

        def post(self,request,format=None):
            data = JSONParser().parse(request)
            serializer = SnippetSerializer(data=data)
            if serializer.is_valid():
                serializer.save()
                return Response(serializer.data, status=201)
            return Response(serializer.errors, status=400)

    class SnippetDetail(APIView):
        def get_object(self, pk):
            try:
                return Snippet.objects.get(pk=pk)
            except Snippet.DoesNotExist:
                return Http404

        def get(self, request,pk,format=None):
            snippet = self.get_object(pk)
            serializer = SnippetSerializer(snippet)
            return Response(serializer.data)

        def put(self, request,pk,format=None):
            snippet = self.get_object(pk)
            serializer = SnippetSerializer(snippet, data=request.data)
            if serializer.is_valid():
                serializer.save()
                return Response(serializer.data)
            return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)

        def delete(self, request,pk,format=None):
            snippet = self.get_object(pk)
            snippet.delete()
            return Response(status=status.HTTP_204_NO_CONTENT)


    #snippets/urls.py
    urlpatterns = [
        url(r'^snippets/$', views.SnippetList.as_view()),
        url(r'^snippets/(?P<pk>[0-9]+)/$', views.SnippetDetail.as_view()),
    ]

####Using Mixins

    from snippets.models import Snippet
    from snippets.serializers import SnippetSerializer
    from rest_framework import mixins, generics

    class SnippetList(mixins.ListModelMixin,
                        mixins.CreateModelMixin,
                        generics.GenericAPIView):
        queryset = Snippet.objects.all()
        serializer_class = SnippetSerializer

        def get(self,request,*args,**kwargs):
            return self.list(request,*args,**kwargs)

        def post(self,request,*args,**kwargs):
            return self.create(request,*args,**kwargs)


    class SnippetDetail(mixins.RetrieveModelMixin,
                        mixins.UpdateModelMixin,
                        mixins.DestroyModelMixin,
                        generics.GenericAPIView):

        queryset = Snippet.objects.all()
        serializer_class = SnippetSerializer

        def get(self,request,*args,**kwargs):
            return self.retrieve(request,*args,**kwargs)

        def put(self,request,*args,**kwargs):
            return self.update(request,*args,**kwargs)

        def delete(self,request,*args,**kwargs):
            return self.delete(request,*args,**kwargs)

####Using generic class-based views

    from snippets.models import Snippet
    from snippets.serializers import SnippetSerializer
    from rest_framework import generics

    class SnippetList(generics.ListCreateAPIView):
        queryset = Snippet.objects.all()
        serializer_class = SnippetSerializer

    class SnippetDetail(generics.RetrieveUpdateDestroyAPIView):
        queryset = Snippet.objects.all()
        serializer_class = SnippetSerializer






##4.Authentication and permissions
##5.Relationships and hyperlinked APIs
##6.Viewsets and routers
##7.Schemas and client libraries
