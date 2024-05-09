### Olá 👋


###### Sobre o Thiago

Profissional em ascensão na área de Análise de Dados pela EBAC, demonstrando um perfil analítico e profundo interesse em tecnologia. Decidiu empreender uma transição de carreira, buscando explorar a compreensão das necessidades das pessoas através da pesquisa e análise de dados. Está atualmente ampliando seu conhecimento e aprimorando seu portfólio com projetos práticos. Possui expertise em:

Python e SQL
Desenvolvimento de soluções analíticas
Estrutura de dados, algoritmos e arquitetura de sistemas
Com uma trajetória em constante evolução, busca oportunidades para aplicar suas habilidades em novos contextos e contribuir de forma significativa para projetos desafiadores.


Uma lista completa das minhas atividades (artigos, podcasts, lives, etc.) está no meu perfil do GitHub, confira: https://github.com/Thiago-Marcelino

Seja bem vindo e fique à vontade para entrar em contato. :)


import {
  clampValue,
  CONSTANTS,
  renderError,
  parseBoolean,
} from "../src/common/utils.js";
import { isLocaleAvailable } from "../src/translations.js";
import { renderGistCard } from "../src/cards/gist-card.js";
import { fetchGist } from "../src/fetchers/gist-fetcher.js";

export default async (req, res) => {
  const {
    id,
    title_color,
    icon_color,
    text_color,
    bg_color,
    theme,
    cache_seconds,
    locale,
    border_radius,
    border_color,
    show_owner,
    hide_border,
  } = req.query;

  res.setHeader("Content-Type", "image/svg+xml");

  if (locale && !isLocaleAvailable(locale)) {
    return res.send(
      renderError("Something went wrong", "Language not found", {
        title_color,
        text_color,
        bg_color,
        border_color,
        theme,
      }),
    );
  }

  try {
    const gistData = await fetchGist(id);

    let cacheSeconds = clampValue(
      parseInt(cache_seconds || CONSTANTS.SIX_HOURS, 10),
      CONSTANTS.SIX_HOURS,
      CONSTANTS.ONE_DAY,
    );
    cacheSeconds = process.env.CACHE_SECONDS
      ? parseInt(process.env.CACHE_SECONDS, 10) || cacheSeconds
      : cacheSeconds;

    /*
      if star count & fork count is over 1k then we are kFormating the text
      and if both are zero we are not showing the stats
      so we can just make the cache longer, since there is no need to frequent updates
    */
    const stars = gistData.starsCount;
    const forks = gistData.forksCount;
    const isBothOver1K = stars > 1000 && forks > 1000;
    const isBothUnder1 = stars < 1 && forks < 1;
    if (!cache_seconds && (isBothOver1K || isBothUnder1)) {
      cacheSeconds = CONSTANTS.SIX_HOURS;
    }

    res.setHeader(
      "Cache-Control",
      `max-age=${
        cacheSeconds / 2
      }, s-maxage=${cacheSeconds}, stale-while-revalidate=${CONSTANTS.ONE_DAY}`,
    );

    return res.send(
      renderGistCard(gistData, {
        title_color,
        icon_color,
        text_color,
        bg_color,
        theme,
        border_radius,
        border_color,
        locale: locale ? locale.toLowerCase() : null,
        show_owner: parseBoolean(show_owner),
        hide_border: parseBoolean(hide_border),
      }),
    );
  } catch (err) {
    res.setHeader(
      "Cache-Control",
      `max-age=${CONSTANTS.ERROR_CACHE_SECONDS / 2}, s-maxage=${
        CONSTANTS.ERROR_CACHE_SECONDS
      }, stale-while-revalidate=${CONSTANTS.ONE_DAY}`,
    ); // Use lower cache period for errors.
    return res.send(
      renderError(err.message, err.secondaryMessage, {
        title_color,
        text_color,
        bg_color,
        border_color,
        theme,
      }),
    );
  }
};
