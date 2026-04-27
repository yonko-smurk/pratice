import os

from django.db import models
from pgvector.django import VectorField

# Embedding dimension. Configurable so the table can be aligned with the
# embedding model in use:
#   * 768  - Ollama nomic-embed-text (default)
#   * 1024 - Ollama mxbai-embed-large
#   * 1536 - OpenAI text-embedding-3-small / ada-002
# Override with EMBED_DIM env var. Changing requires a migration + re-ingest.
EMBED_DIM = int(os.getenv("EMBED_DIM", "768"))


class KbArticle(models.Model):
    title = models.CharField(max_length=200, unique=True)
    body_md = models.TextField()
    is_published = models.BooleanField(default=True)
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)

    def __str__(self):
        return self.title


class KbChunk(models.Model):
    article = models.ForeignKey(
        KbArticle, on_delete=models.CASCADE, related_name="chunks"
    )
    chunk_index = models.PositiveIntegerField()
    text = models.TextField()
    embedding = VectorField(dimensions=EMBED_DIM, null=True, blank=True)
    token_count = models.PositiveIntegerField(default=0)
    created_at = models.DateTimeField(auto_now_add=True)

    class Meta:
        ordering = ["article_id", "chunk_index"]
        unique_together = ("article", "chunk_index")

    def __str__(self):
        return f"{self.article.title} [{self.chunk_index}]"
