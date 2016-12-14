##1.Serialization
#### Model -> Serializer -> JSONRenderer
- snippet = Snippet(code='print hello, world')
- serializer = SnippetSerializer(snippet)
- content = JSONRenderer().render(serializer.data)

#### Stream -> JSONParser -> Serializers -> Model
- stream = BytesIO(content)
- data = JSONParser().parse(stream)
- serializer = SnippetSerializer(data=data)
- serializer.is_valid() && serializer.save()

##2.Requests and responses

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

    class SnippetList(generics.ListCreateAPIView):
        queryset = Snippet.objects.all()
        serializer_class = SnippetSerializer

    class SnippetDetail(generics.RetrieveUpdateDestroyAPIView):
        queryset = Snippet.objects.all()
        serializer_class = SnippetSerializer


##4.Authentication and permissions
####Add User Serializer and Snippet model attribute

    #snippets/models.py
    owner = models.ForeignKey('auth.User', related_name='snippets', on_delete=models.CASCADE)

    #snippets/views.py
    class UserList(generics.ListAPIView):
    class UserDetail(generics.RetrieveAPIView):

####Associating Snippets With Users
Associating the user that created the snippet. The user isn't sent as part of the serialized representation, but is instead a property of the incoming request.

Overriding a .perform_create() allows us to modify how the instance save is managed, and handle any information that is implicit in the incoming request or requested URL.

    #snippets/views.py
    class SnippetList(generics.ListCreateAPIView):
        queryset = Snippet.objects.all()
        serializer_class = SnippetSerializer

        def perform_create(self,serializer):
            serializer.save(owner=self.request.user)

    #snippets/serializers.py
    class SnippetSerializer(serializers.ModelSerializer):
        owner = serializers.ReadOnlyField(source='owner.username')

The create() method of our serializer will now be passed an additional 'owner' field, along with the validated data from the request.

####You can also customize Permissions

    #snippets/permissions.py
    class IsOwnerOrReadOnly(permissions.BasePermission):

        def has_object_permission(self, request, view, obj):
            # Read permissions are allowed to any request,
            # so we'll always allow GET, HEAD or OPTIONS requests.
            if request.method in permissions.SAFE_METHODS:
                return True

            # Write permissions are only allowed to the owner of the snippet.
            return obj.owner == request.user

    #snippets/views.py
    class SnippetDetail(generics.RetrieveUpdateDestroyAPIView):
        permission_classes = (permissions.IsAuthenticatedOrReadOnly, IsOwnerOrReadOnly,)

####Httpie test request
- http POST http://127.0.0.1:8000/snippets/ code="print(123)" # it will return authentication error
- python manage.py createsuperuser
- http -a tom:password123 POST http://127.0.0.1:8000/snippets/ code="print(789)"

##5.Relationships and hyperlinked APIs


####Reverse Data to Url  

    #snippets/views.py
    @api_view(['GET'])
    def api_root(request, format=None):
        return Response({
            'users': reverse('user-list', request=request, format=format),
            'snippets': reverse('snippet-list', request=request, format=format)
        })

    #snippets/urls.py
    url(r'^$', views.api_root),

    #result
    {
        "snippets": "http://127.0.0.1:8000/snippets/",
        "users": "http://127.0.0.1:8000/users/"
    }

####HyperlinkedAPI Process
Basically, entities represent relationship with other entity as number of ForeignKey.
On the contrary to this, HyperlinkedAPI show its url in the form of RESTful API.

    #snippets/views.py
    class UserSerializer(serializers.HyperlinkedModelSerializer):
        snippets = serializers.HyperlinkedRelatedField(many=True,view_name="snippet-detail",read_only=True)

    #snippets/urls.py
    url(r'^snippets/(?P<pk>[0-9]+)/$', views.SnippetDetail.as_view(), name="snippet-detail"),

    #result
    {
        "id": 3,
        "username": "tom",
        "snippets": [
            "http://127.0.0.1:8000/snippets/1/",
            "http://127.0.0.1:8000/snippets/2/"
        ]
    }

##6.Viewsets and routers

####Apply Django REST frameworks ViewSets

ViewSets allows the developer to leave the URL construction to be handled automatically, based on common conventions.

    #snippets/views.py
    #after - User
    class UserViewSet(viewsets.ReadOnlyModelViewSet):
        queryset = User.objects.all()
        serializer_class = UserSerializer

    #after - Snippet
    class SnippetViewSet(viewsets.ModelViewSet):
        queryset = Snippet.objects.all()
        serializer_class = SnippetSerializer
        permission_classes = (permissions.IsAuthenticatedOrReadOnly, IsOwnerOrReadOnly)

        #Follow related entity along foreignKey
        def perform_create(self, serializer):
            serializer.save(owner = self.request.user)

        #the name of function would be url /snippet/1/highlight/ (not sure..)
        @detail_route(renderer_classes = [renderers.StaticHTMLRenderer])
        def highlight(self, request, *args, **kwargs):
            snippet = self.get_object()
            return Response(snippet.highlighted)

####Router also design the URL conf automatically

    # Create a router and register our viewsets with it.
    router = DefaultRouter()
    router.register(r'snippets', views.SnippetViewSet)
    router.register(r'users', views.UserViewSet)

    # The API URLs are now determined automatically by the router.
    # Additionally, we include the login URLs for the browsable API.
    urlpatterns = [
        url(r'^', include(router.urls)),
        url(r'^api-auth/', include('rest_framework.urls', namespace='rest_framework'))
    ]
