---
title: "Máquinas"
layout: "list"  # Asegúrate de que el layout sea el correcto, puede ser 'list' o cualquier otro que estés usando.
---

{{ range .Pages }}
  {{ if .Params.image }}
    <div class="article-image">
      <img src="{{ .Params.image }}" alt="{{ .Title }}" class="featured-image" />
    </div>
  {{ end }}
  <h2><a href="{{ .Permalink }}">{{ .Title }}</a></h2>
  <p>{{ .Summary }}</p>  <!-- Si quieres mostrar un resumen o descripción breve -->
{{ end }}

