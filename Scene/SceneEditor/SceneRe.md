 ~~~C++
 	void SceneSolutionResourceMetadataWriter<Writer>::Write( Writer& writer ){...	

	res::ResourceTokenHandle token = GResourceLoader->IssueLoadingRequest( Convert( m_resourcePath, options ) );//资源加载
       
		THandle<tools::SceneEditorResource> sceneEditorResource = Cast<tools::SceneEditorResource>( token->WaitUntilLoaded() );

		if ( sceneEditorResource == nullptr )
		{
			writer.Null();
			return;
		}//获取SceneResource

		const THandle< tools::SceneBackendData>& sceneBackendData = sceneEditorResource->GetBackendData();
		if ( sceneBackendData == nullptr )
		{
			writer.Null();
			return;
		}//获取SceneBackendData


        red::HashMap<CName, int> nodeTypeCount;
			const Bool ret = sceneBackendData->m_graphDescriptorGlobalData->Initialize( sceneEditorResource );
			if ( ret )
			{
				sceneBackendData->m_graphDescriptor->m_sceneDescriptor = sceneEditorResource->m_sceneDescriptor.Get();
				sceneBackendData->m_graphDescriptor->PropagateGlobalData( sceneBackendData->m_graphDescriptorGlobalData );
				sceneBackendData->m_graphDescriptor->Initialize( sceneEditorResource );

				sceneBackendData->m_graphDescriptor->IterateNodes( [&]( tools::IGraphNodeDescriptor* baseGraphnode )
				{
					if ( THandle<scnb::graph::SceneGraphNodeDescriptor> graphnode = Cast<scnb::graph::SceneGraphNodeDescriptor>( HandleFromPtr( baseGraphnode ) ) )
					{
						if ( graphnode == nullptr || graphnode->GetGraphDescriptor() == nullptr )
						{
							return;
						}

						auto& node = graphnode->GetNode();
						CName nodeTypeName = node.GetClass()->GetName();

						// Count node types
						auto it = nodeTypeCount.Find( nodeTypeName );
						if ( it == nodeTypeCount.End() )
						{
							nodeTypeCount[ nodeTypeName ] = 1;
						}
						else
						{
							++it.Value();
						}
					}
				}, []( tools::IGraphNodeDescriptor* baseGraphnode ) {} );
			}

...}

~~~
先通过red::DynArray< THandle< tools::NodeDescriptor > > sectionNodes;拿到所有SectionNode 
~~~C++

const red::DynArray< THandle<tools::EventDescriptor> >& sectionEvents = sectionNode->GetEvents();
				for ( const auto& event : sectionEvents )
				{
					if ( event->IsA<tools::PlaySkAnimDescriptor>() )
					{
						THandle<tools::PlaySkAnimDescriptor> skeletalAnimDescriptor = Cast<tools::PlaySkAnimDescriptor>( event );
						AddAnimationToMap( sectionAnimations, skeletalAnimDescriptor->m_animName );
						AddAnimationToMap( PlaySkAnimNames, skeletalAnimDescriptor->m_animName );
					}
					else if ( event->IsA<tools::ChangeIdleAnimDescriptor>() )
					{
						THandle<tools::ChangeIdleAnimDescriptor> changeIdleAnimDescriptor = Cast<tools::ChangeIdleAnimDescriptor>( event );
						AddAnimationToMap( sectionAnimations, changeIdleAnimDescriptor->m_animName );
						AddAnimationToMap( sectionAnimations, changeIdleAnimDescriptor->m_idleAnimName );
						AddAnimationToMap( sectionAnimations, changeIdleAnimDescriptor->m_addIdleAnimName );
						AddAnimationToMap( ChangeIdleAnimNames, changeIdleAnimDescriptor->m_animName );
						AddAnimationToMap( ChangeIdleAnimNames, changeIdleAnimDescriptor->m_idleAnimName );
						AddAnimationToMap( ChangeIdleAnimNames, changeIdleAnimDescriptor->m_addIdleAnimName );
						if ( changeIdleAnimDescriptor->m_customTransitionAnim != CName::NONE() )
						{
							AddAnimationToMap( sectionAnimations, changeIdleAnimDescriptor->m_customTransitionAnim );
							AddAnimationToMap( ChangeIdleAnimNames, changeIdleAnimDescriptor->m_customTransitionAnim );
						}
					}
					else if ( event->IsA<scnb::events::ChangeWorkEvent>() )
					{
						THandle<scnb::events::ChangeWorkEvent> changeWorkDescriptor = Cast<scnb::events::ChangeWorkEvent>( event );
						if ( changeWorkDescriptor->GetTransitionAnimInfo().m_animName != CName::NONE() )
						{
							AddAnimationToMap( sectionAnimations, changeWorkDescriptor->GetTransitionAnimInfo().m_animName );
							AddAnimationToMap( ChangeWorkAnimNames, changeWorkDescriptor->GetTransitionAnimInfo().m_animName );
						}
						if ( changeWorkDescriptor->GetGameplayAnimInfo().m_animName != CName::NONE() )
						{
							AddAnimationToMap( sectionAnimations, changeWorkDescriptor->GetGameplayAnimInfo().m_animName );
							AddAnimationToMap( ChangeWorkAnimNames, changeWorkDescriptor->GetGameplayAnimInfo().m_animName );
						}
					}
					else if ( event->IsA<scnb::events::StopWorkEvent>() )
					{
						THandle<scnb::events::StopWorkEvent> stopWorkDescriptor = Cast<scnb::events::StopWorkEvent>( event );
						if ( stopWorkDescriptor->GetAnimationInfo().m_animName != CName::NONE() )
						{
							AddAnimationToMap( sectionAnimations, stopWorkDescriptor->GetAnimationInfo().m_animName );
							AddAnimationToMap( StopWorkAnimNames, stopWorkDescriptor->GetAnimationInfo().m_animName );
						}
						if ( stopWorkDescriptor->GetGameplayAnimInfo().m_animName != CName::NONE() )
						{
							AddAnimationToMap( sectionAnimations, stopWorkDescriptor->GetGameplayAnimInfo().m_animName );
							AddAnimationToMap( StopWorkAnimNames, stopWorkDescriptor->GetGameplayAnimInfo().m_animName );
						}
					}
				}
~~~
通过const red::DynArray< THandle<tools::EventDescriptor> >& sectionEvents = sectionNode->GetEvents();方式获取所有Clip Event种类
