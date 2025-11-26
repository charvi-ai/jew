import React, { useState, useEffect } from 'react';
import { base44 } from '@/api/base44Client';
import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query';
import { Card, CardContent, CardHeader, CardTitle } from "@/components/ui/card";
import { Button } from "@/components/ui/button";
import { Tabs, TabsContent, TabsList, TabsTrigger } from "@/components/ui/tabs";
import { Loader2, Sparkles, Film, Music, Heart, TrendingUp } from "lucide-react";
import GenreSelector from '@/components/recommendations/GenreSelector';
import FavoritesInput from '@/components/recommendations/FavoritesInput';
import RecommendationCard from '@/components/recommendations/RecommendationCard';

export default function Home() {
  const [activeTab, setActiveTab] = useState('discover');
  const [selectedType, setSelectedType] = useState('movie');
  const [movieGenres, setMovieGenres] = useState([]);
  const [musicGenres, setMusicGenres] = useState([]);
  const [favoriteMovies, setFavoriteMovies] = useState([]);
  const [favoriteArtists, setFavoriteArtists] = useState([]);
  const [isGenerating, setIsGenerating] = useState(false);
  const [recommendations, setRecommendations] = useState([]);

  const queryClient = useQueryClient();

  const { data: savedRecommendations = [], isLoading } = useQuery({
    queryKey: ['recommendations'],
    queryFn: () => base44.entities.Recommendation.list('-created_date'),
  });

  const { data: preferences = [] } = useQuery({
    queryKey: ['preferences'],
    queryFn: () => base44.entities.UserPreference.list(),
  });

  useEffect(() => {
    if (preferences.length > 0) {
      const moviePref = preferences.find(p => p.preference_type === 'movie');
      const musicPref = preferences.find(p => p.preference_type === 'music');
      if (moviePref) {
        setMovieGenres(moviePref.genres || []);
        setFavoriteMovies(moviePref.favorite_items || []);
      }
      if (musicPref) {
        setMusicGenres(musicPref.genres || []);
        setFavoriteArtists(musicPref.favorite_items || []);
      }
    }
  }, [preferences]);

  const saveMutation = useMutation({
    mutationFn: async (item) => {
      if (item.id) {
        return base44.entities.Recommendation.update(item.id, { is_saved: !item.is_saved });
      } else {
        return base44.entities.Recommendation.create({ ...item, is_saved: true });
      }
    },
    onSuccess: () => queryClient.invalidateQueries({ queryKey: ['recommendations'] }),
  });

  const rateMutation = useMutation({
    mutationFn: async ({ item, rating }) => {
      if (item.id) {
        return base44.entities.Recommendation.update(item.id, { rating });
      } else {
        return base44.entities.Recommendation.create({ ...item, rating });
      }
    },
    onSuccess: () => queryClient.invalidateQueries({ queryKey: ['recommendations'] }),
  });

  const generateRecommendations = async () => {
    setIsGenerating(true);
    
    const genres = selectedType === 'movie' ? movieGenres : musicGenres;
    const favorites = selectedType === 'movie' ? favoriteMovies : favoriteArtists;
    
    const prompt = selectedType === 'movie'
      ? `Generate 6 movie recommendations based on these preferences:
         Favorite genres: ${genres.join(', ') || 'Any'}
         Favorite movies: ${favorites.join(', ') || 'None specified'}
         
         For each movie, provide: title, director, genre, year, and a brief 2-sentence description.
         Make recommendations diverse and include both popular and hidden gems.`
      : `Generate 6 music recommendations (songs or albums) based on these preferences:
         Favorite genres: ${genres.join(', ') || 'Any'}
         Favorite artists: ${favorites.join(', ') || 'None specified'}
         
         For each recommendation, provide: title (song/album name), artist, genre, year, and a brief 2-sentence description.
         Make recommendations diverse and include both popular and underground artists.`;

    const response = await base44.integrations.Core.InvokeLLM({
      prompt,
      response_json_schema: {
        type: "object",
        properties: {
          recommendations: {
            type: "array",
            items: {
              type: "object",
              properties: {
                title: { type: "string" },
                artist_director: { type: "string" },
                genre: { type: "string" },
                year: { type: "string" },
                description: { type: "string" }
              }
            }
          }
        }
      }
    });

    const newRecs = response.recommendations.map(rec => ({
      ...rec,
      recommendation_type: selectedType,
      is_saved: false,
      rating: 0
    }));

    setRecommendations(newRecs);
    setIsGenerating(false);
  };

  const toggleGenre = (genre, type) => {
    if (type === 'movie') {
      setMovieGenres(prev => 
        prev.includes(genre) ? prev.filter(g => g !== genre) : [...prev, genre]
      );
    } else {
      setMusicGenres(prev => 
        prev.includes(genre) ? prev.filter(g => g !== genre) : [...prev, genre]
      );
    }
  };

  const savedItems = savedRecommendations.filter(r => r.is_saved);
  const displayRecs = activeTab === 'saved' 
    ? savedItems 
    : recommendations.length > 0 
      ? recommendations 
      : savedRecommendations.slice(0, 6);

  return (
    <div className="min-h-screen bg-gradient-to-br from-gray-900 via-purple-900 to-gray-900">
      {/* Header */}
      <div className="relative overflow-hidden">
        <div className="absolute inset-0 bg-[url('https://images.unsplash.com/photo-1489599849927-2ee91cede3ba?w=1920')] bg-cover bg-center opacity-20" />
        <div className="relative px-6 py-12 text-center">
          <div className="flex items-center justify-center gap-3 mb-4">
            <Sparkles className="w-10 h-10 text-purple-400" />
            <h1 className="text-4xl md:text-5xl font-bold text-white">
              AI Entertainment Hub
            </h1>
          </div>
          <p className="text-gray-300 text-lg max-w-2xl mx-auto">
            Discover personalized movie and music recommendations powered by AI
          </p>
        </div>
      </div>

      <div className="max-w-7xl mx-auto px-6 py-8">
        {/* Main Tabs */}
        <Tabs value={activeTab} onValueChange={setActiveTab} className="space-y-6">
          <TabsList className="bg-white/10 backdrop-blur-sm p-1">
            <TabsTrigger value="discover" className="data-[state=active]:bg-purple-600 text-white">
              <TrendingUp className="w-4 h-4 mr-2" />
              Discover
            </TabsTrigger>
            <TabsTrigger value="preferences" className="data-[state=active]:bg-purple-600 text-white">
              <Sparkles className="w-4 h-4 mr-2" />
              Preferences
            </TabsTrigger>
            <TabsTrigger value="saved" className="data-[state=active]:bg-purple-600 text-white">
              <Heart className="w-4 h-4 mr-2" />
              Saved ({savedItems.length})
            </TabsTrigger>
          </TabsList>

          {/* Discover Tab */}
          <TabsContent value="discover" className="space-y-6">
            {/* Type Selector */}
            <div className="flex gap-4 justify-center">
              <Button
                variant={selectedType === 'movie' ? 'default' : 'outline'}
                onClick={() => setSelectedType('movie')}
                className={selectedType === 'movie' 
                  ? 'bg-blue-600 hover:bg-blue-700' 
                  : 'bg-white/10 text-white border-white/20 hover:bg-white/20'
                }
              >
                <Film className="w-4 h-4 mr-2" />
                Movies
              </Button>
              <Button
                variant={selectedType === 'music' ? 'default' : 'outline'}
                onClick={() => setSelectedType('music')}
                className={selectedType === 'music' 
                  ? 'bg-pink-600 hover:bg-pink-700' 
                  : 'bg-white/10 text-white border-white/20 hover:bg-white/20'
                }
              >
                <Music className="w-4 h-4 mr-2" />
                Music
              </Button>
            </div>

            {/* Generate Button */}
            <div className="text-center">
              <Button
                onClick={generateRecommendations}
                disabled={isGenerating}
                size="lg"
                className="bg-gradient-to-r from-purple-600 to-pink-600 hover:from-purple-700 hover:to-pink-700 text-white px-8"
              >
                {isGenerating ? (
                  <>
                    <Loader2 className="w-5 h-5 mr-2 animate-spin" />
                    Generating...
                  </>
                ) : (
                  <>
                    <Sparkles className="w-5 h-5 mr-2" />
                    Get AI Recommendations
                  </>
                )}
              </Button>
            </div>

            {/* Recommendations Grid */}
            {isLoading ? (
              <div className="flex justify-center py-12">
                <Loader2 className="w-8 h-8 animate-spin text-purple-400" />
              </div>
            ) : displayRecs.length > 0 ? (
              <div className="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-6">
                {displayRecs
                  .filter(r => activeTab === 'saved' || r.recommendation_type === selectedType)
                  .map((item, index) => (
                    <RecommendationCard
                      key={item.id || index}
                      item={item}
                      onSave={(item) => saveMutation.mutate(item)}
                      onRate={(item, rating) => rateMutation.mutate({ item, rating })}
                    />
                  ))}
              </div>
            ) : (
              <Card className="bg-white/10 backdrop-blur-sm border-white/20">
                <CardContent className="py-12 text-center">
                  <Sparkles className="w-12 h-12 text-purple-400 mx-auto mb-4" />
                  <p className="text-white text-lg">
                    Click "Get AI Recommendations" to discover personalized suggestions!
                  </p>
                  <p className="text-gray-400 mt-2">
                    Set your preferences first for better results
                  </p>
                </CardContent>
              </Card>
            )}
          </TabsContent>

          {/* Preferences Tab */}
          <TabsContent value="preferences" className="space-y-6">
            <div className="grid md:grid-cols-2 gap-6">
              {/* Movie Preferences */}
              <Card className="bg-white/10 backdrop-blur-sm border-white/20">
                <CardHeader>
                  <CardTitle className="text-white flex items-center gap-2">
                    <Film className="w-5 h-5 text-blue-400" />
                    Movie Preferences
                  </CardTitle>
                </CardHeader>
                <CardContent className="space-y-4">
                  <div>
                    <label className="text-gray-300 text-sm font-medium mb-2 block">
                      Favorite Genres
                    </label>
                    <GenreSelector
                      type="movie"
                      selectedGenres={movieGenres}
                      onToggle={(genre) => toggleGenre(genre, 'movie')}
                    />
                  </div>
                  <div>
                    <label className="text-gray-300 text-sm font-medium mb-2 block">
                      Favorite Movies
                    </label>
                    <FavoritesInput
                      type="movie"
                      favorites={favoriteMovies}
                      onAdd={(movie) => setFavoriteMovies(prev => [...prev, movie])}
                      onRemove={(movie) => setFavoriteMovies(prev => prev.filter(m => m !== movie))}
                      placeholder="Add a favorite movie..."
                    />
                  </div>
                </CardContent>
              </Card>

              {/* Music Preferences */}
              <Card className="bg-white/10 backdrop-blur-sm border-white/20">
                <CardHeader>
                  <CardTitle className="text-white flex items-center gap-2">
                    <Music className="w-5 h-5 text-pink-400" />
                    Music Preferences
                  </CardTitle>
                </CardHeader>
                <CardContent className="space-y-4">
                  <div>
                    <label className="text-gray-300 text-sm font-medium mb-2 block">
                      Favorite Genres
                    </label>
                    <GenreSelector
                      type="music"
                      selectedGenres={musicGenres}
                      onToggle={(genre) => toggleGenre(genre, 'music')}
                    />
                  </div>
                  <div>
                    <label className="text-gray-300 text-sm font-medium mb-2 block">
                      Favorite Artists
                    </label>
                    <FavoritesInput
                      type="music"
                      favorites={favoriteArtists}
                      onAdd={(artist) => setFavoriteArtists(prev => [...prev, artist])}
                      onRemove={(artist) => setFavoriteArtists(prev => prev.filter(a => a !== artist))}
                      placeholder="Add a favorite artist..."
                    />
                  </div>
                </CardContent>
              </Card>
            </div>
          </TabsContent>

          {/* Saved Tab */}
          <TabsContent value="saved">
            {savedItems.length > 0 ? (
              <div className="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-6">
                {savedItems.map((item) => (
                  <RecommendationCard
                    key={item.id}
                    item={item}
                    onSave={(item) => saveMutation.mutate(item)}
                    onRate={(item, rating) => rateMutation.mutate({ item, rating })}
                  />
                ))}
              </div>
            ) : (
              <Card className="bg-white/10 backdrop-blur-sm border-white/20">
                <CardContent className="py-12 text-center">
                  <Heart className="w-12 h-12 text-gray-500 mx-auto mb-4" />
                  <p className="text-white text-lg">No saved items yet</p>
                  <p className="text-gray-400 mt-2">
                    Click the heart icon on recommendations to save them
                  </p>
                </CardContent>
              </Card>
            )}
          </TabsContent>
        </Tabs>
      </div>
    </div>
  );
}
