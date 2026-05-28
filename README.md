Climate Granularity, Part 1
================

- [Purpose](#purpose)
- [Model core](#model-core)
- [Generate the four central
  scenarios](#generate-the-four-central-scenarios)
- [Figure helpers](#figure-helpers)
- [Figure 1: relative premiums](#figure-1-relative-premiums)
- [Figure 2: market shares by risk
  class](#figure-2-market-shares-by-risk-class)
- [Figure 3: portfolio composition](#figure-3-portfolio-composition)
- [Figure 4: loss ratios](#figure-4-loss-ratios)

# Purpose

This notebook is intentionally minimal. It rebuilds the four central
scenarios and produces only the four figures used in the paper:

1.  relative premium paths by insurer and scenario;
2.  market shares by risk class and scenario;
3.  portfolio composition by insurer and scenario;
4.  loss ratios by risk class and scenario.

The notebook is self-contained: it does not call `source()` and does not
use `find_script()`.

# Model core

``` r
run_market_simulation <- function(
  scenario = c("unconstrained", "national", "commune", "risk_class"),
  seed = 123,
  I = 20,
  N_POP = 500,
  DIRICHLET_ALPHA = rep(1, 3),
  RISK_PROB = c(0.1, 0.6, 0.9),
  RISK_COST_PROB = c(0.2, 0.5, 0.8),
  ALPHA = 10.0,
  BETA_PRICE = 1.0,
  V_INSURER = c(A = 0.0, B = 0.0, C = 0.0),
  ETA_CAT = 1.0,
  MS_CONSTRAINT = 0.10,
  DAMPING_NU = 1.0,
  DENOM_EPSILON = 0.01,
  PRICE_LB = 0.01,
  PRICE_UB = ALPHA,
  CHARGE_FACTOR = 1.02,
  T_MAX = 39,
  T_INSPECT = 35,
  quiet = TRUE
) {
  scenario <- match.arg(scenario)

  if (scenario == "unconstrained") {
    # Same optimizer and local constraint block as the base script, but with a zero floor.
    constraint_type <- "commune"
    MS_CONSTRAINT <- 1.0
  } else if (scenario == "national") {
    constraint_type <- "national"
  } else if (scenario == "commune") {
    constraint_type <- "commune"
  } else if (scenario == "risk_class") {
    constraint_type <- "risk_class"
  }

  ASSURERS_CONFIG <- list(
    A = list(archetype = "national", activation_active = TRUE, t_start = 2,
             t_change = NULL, new_archetype = NULL, ms_constraint = MS_CONSTRAINT),
    B = list(archetype = "communal", activation_active = TRUE, t_start = 2,
             t_change = NULL, new_archetype = NULL, ms_constraint = MS_CONSTRAINT),
    C = list(archetype = "communal_risk", activation_active = TRUE, t_start = 1,
             t_change = NULL, new_archetype = NULL, ms_constraint = MS_CONSTRAINT)
  )

  get_v_insurer <- function(name) {
    if (!is.null(names(V_INSURER)) && name %in% names(V_INSURER)) return(V_INSURER[[name]])
    0.0
  }

  rdirichlet <- function(n, alpha) {
    k <- length(alpha)
    x <- matrix(
      rgamma(n * k, shape = rep(alpha, each = n), rate = 1),
      nrow = n,
      ncol = k,
      byrow = FALSE
    )
    x / rowSums(x)
  }

  set.seed(seed)
  N <- rep(N_POP, I)
  risk_props <- rdirichlet(I, DIRICHLET_ALPHA)

  risk_prob <- RISK_PROB
  risk_cost_prob <- RISK_COST_PROB
  alpha <- ALPHA

  risk_cost <- ETA_CAT * alpha * risk_cost_prob
  comm_cost <- as.vector(risk_props %*% (risk_cost * risk_prob))
  u <- as.vector(risk_props %*% risk_prob)

  get_archetype_shape <- function(archetype_key, I_communes = I) {
    if (archetype_key == "national") return(1)
    if (archetype_key == "communal") return(I_communes)
    if (archetype_key == "communal_risk") return(c(I_communes, 3))
    stop(sprintf("Unknown archetype: %s", archetype_key))
  }

  reshape_price_to_standard <- function(price, archetype_key, I_communes = I) {
    if (archetype_key == "national") {
      return(matrix(as.numeric(price)[1], nrow = I_communes, ncol = 3))
    }
    if (archetype_key == "communal") {
      price_vec <- as.numeric(price)
      if (length(price_vec) != I_communes) stop("Invalid communal price vector length.")
      return(matrix(rep(price_vec, each = 3), nrow = I_communes, ncol = 3, byrow = TRUE))
    }
    if (archetype_key == "communal_risk") {
      if (is.matrix(price) && all(dim(price) == c(I_communes, 3))) return(price)
      price_vec <- as.numeric(price)
      if (length(price_vec) != I_communes * 3) stop("Invalid communal-risk price vector length.")
      return(matrix(price_vec, nrow = I_communes, ncol = 3, byrow = TRUE))
    }
    stop(sprintf("Unknown archetype: %s", archetype_key))
  }

  reshape_price_from_standard <- function(price_matrix, archetype_key, I_communes = I) {
    price_matrix <- matrix(as.numeric(price_matrix), nrow = I_communes, ncol = 3)
    if (archetype_key == "national") return(mean(price_matrix))
    if (archetype_key == "communal") return(rowMeans(price_matrix))
    if (archetype_key == "communal_risk") return(as.vector(t(price_matrix)))
    stop(sprintf("Unknown archetype: %s", archetype_key))
  }

  archetype_vector_to_object <- function(price_flat, archetype_key, I_communes = I) {
    if (archetype_key == "national") return(as.numeric(price_flat)[1])
    if (archetype_key == "communal") return(as.numeric(price_flat))
    if (archetype_key == "communal_risk") {
      return(matrix(as.numeric(price_flat), nrow = I_communes, ncol = 3, byrow = TRUE))
    }
    stop(sprintf("Unknown archetype: %s", archetype_key))
  }

  standardize_price_input <- function(price, name = "") {
    p <- as.numeric(price)
    if (length(p) == 1) return(matrix(p[1], nrow = I, ncol = 3))
    if (is.matrix(price) && all(dim(price) == c(I, 3))) return(price)
    if (length(p) == I * 3) return(matrix(p, nrow = I, ncol = 3, byrow = TRUE))
    stop(sprintf("Invalid price shape for %s", name))
  }

  shares_infr_comm <- function(prices_dict) {
    prices_std <- list()
    for (name in names(prices_dict)) {
      prices_std[[name]] <- standardize_price_input(prices_dict[[name]], name)
    }

    sorted_names <- sort(names(prices_std))
    alpha_u <- matrix(alpha * u, nrow = I, ncol = 3)

    exp_utils_list <- lapply(sorted_names, function(name) {
      utilities <- alpha_u + get_v_insurer(name) - BETA_PRICE * prices_std[[name]]
      utilities <- pmin(pmax(utilities, -500.0), 400.0)
      exp(utilities)
    })

    denom <- Reduce("+", exp_utils_list) + DENOM_EPSILON

    result <- list()
    for (idx in seq_along(sorted_names)) {
      name <- sorted_names[[idx]]
      result[[name]] <- exp_utils_list[[idx]] / denom
    }
    result
  }

  shares_comm <- function(prices_dict) {
    shares_infr <- shares_infr_comm(prices_dict)
    lapply(shares_infr, function(mat) rowSums(mat * risk_props))
  }

  shares <- function(prices_dict) {
    shares_c <- shares_comm(prices_dict)
    lapply(shares_c, mean)
  }

  shares_by_risk_class <- function(prices_dict, insurer_name) {
    share_mat <- shares_infr_comm(prices_dict)[[insurer_name]]
    risk_weights <- sweep(risk_props, 1, N, `*`)
    vapply(seq_len(ncol(risk_weights)), function(rr) {
      sum(risk_weights[, rr] * share_mat[, rr]) / sum(risk_weights[, rr])
    }, numeric(1))
  }

  primepure <- function() {
    sum(N * (comm_cost * u)) / sum(N)
  }

  profit <- function(assurer_name, prices_std_dict) {
    shares_infr <- shares_infr_comm(prices_std_dict)
    share_assurer <- shares_infr[[assurer_name]]
    N_risk <- matrix(N, nrow = I, ncol = 3) * risk_props
    pure_ir <- outer(u, risk_cost * risk_prob)
    profit_infr <- (share_assurer * N_risk) *
      (prices_std_dict[[assurer_name]] - pure_ir)
    sum(profit_infr)
  }

  profit_neg <- function(assurer_name, prices_std_dict) {
    -profit(assurer_name, prices_std_dict)
  }

  init_prices_for_assurer <- function(assurer_name) {
    archetype <- ASSURERS_CONFIG[[assurer_name]]$archetype
    pp_base <- primepure() * CHARGE_FACTOR
    if (archetype == "national") return(pp_base)
    if (archetype == "communal") return(rep(pp_base, I))
    if (archetype == "communal_risk") return(matrix(pp_base, nrow = I, ncol = 3))
    stop(sprintf("Unknown archetype: %s", archetype))
  }

  get_bounds_for_assurer <- function(assurer_name, archetype_key = NULL) {
    if (is.null(archetype_key)) archetype_key <- ASSURERS_CONFIG[[assurer_name]]$archetype
    n_params <- prod(get_archetype_shape(archetype_key))
    list(lower = rep(PRICE_LB, n_params), upper = rep(PRICE_UB, n_params))
  }

  prices_opts <- list()
  for (name in names(ASSURERS_CONFIG)) {
    prices_opts[[name]] <- list(init_prices_for_assurer(name))
  }

  archetype_history <- lapply(ASSURERS_CONFIG, function(cfg) cfg$archetype)
  prices_std_history <- list()
  for (name in names(ASSURERS_CONFIG)) {
    cfg <- ASSURERS_CONFIG[[name]]
    prices_std_history[[name]] <- list(
      reshape_price_to_standard(prices_opts[[name]][[1]], cfg$archetype)
    )
  }

  for (t in seq_len(T_MAX)) {
    current_archetype <- list()
    for (name in names(ASSURERS_CONFIG)) {
      cfg <- ASSURERS_CONFIG[[name]]
      if (!is.null(cfg$t_change) && t >= cfg$t_change) {
        current_archetype[[name]] <- cfg$new_archetype
        archetype_history[[name]] <- c(archetype_history[[name]], cfg$new_archetype)
      } else {
        current_archetype[[name]] <- cfg$archetype
        archetype_history[[name]] <- c(archetype_history[[name]], cfg$archetype)
      }
    }

    prices_std_prev <- list()
    for (name in names(ASSURERS_CONFIG)) {
      prices_std_prev[[name]] <- prices_std_history[[name]][[t]]
    }
    prices_std_new <- lapply(prices_std_prev, function(x) x)

    for (name in sort(names(ASSURERS_CONFIG))) {
      cfg <- ASSURERS_CONFIG[[name]]
      if (!cfg$activation_active || t < cfg$t_start) next

      prices_others_std <- prices_std_prev[names(prices_std_prev) != name]

      profit_partial <- function(price_flat) {
        prices_test <- prices_others_std
        price_obj <- archetype_vector_to_object(price_flat, current_archetype[[name]])
        prices_test[[name]] <- reshape_price_to_standard(price_obj, current_archetype[[name]])
        profit_neg(name, prices_test)
      }

      if (constraint_type == "national") {
        share_prev <- shares(prices_std_prev)[[name]]
        share_floor <- share_prev * (1 - cfg$ms_constraint)
        constraint_market_share <- function(price_flat) {
          prices_test <- prices_others_std
          price_obj <- archetype_vector_to_object(price_flat, current_archetype[[name]])
          prices_test[[name]] <- reshape_price_to_standard(price_obj, current_archetype[[name]])
          share_test <- shares(prices_test)[[name]]
          share_test - share_floor
        }
      } else if (constraint_type == "risk_class") {
        share_prev <- shares_by_risk_class(prices_std_prev, name)
        share_floor <- share_prev * (1 - cfg$ms_constraint)
        constraint_market_share <- function(price_flat) {
          prices_test <- prices_others_std
          price_obj <- archetype_vector_to_object(price_flat, current_archetype[[name]])
          prices_test[[name]] <- reshape_price_to_standard(price_obj, current_archetype[[name]])
          share_test <- shares_by_risk_class(prices_test, name)
          share_test - share_floor
        }
      } else {
        share_prev <- shares_comm(prices_std_prev)[[name]]
        share_floor <- share_prev * (1 - cfg$ms_constraint)
        constraint_market_share <- function(price_flat) {
          prices_test <- prices_others_std
          price_obj <- archetype_vector_to_object(price_flat, current_archetype[[name]])
          prices_test[[name]] <- reshape_price_to_standard(price_obj, current_archetype[[name]])
          share_test <- shares_comm(prices_test)[[name]]
          share_test - share_floor
        }
      }

      bounds <- get_bounds_for_assurer(name, current_archetype[[name]])
      price_init <- as.numeric(reshape_price_from_standard(prices_std_prev[[name]], current_archetype[[name]]))

      slsqp_args <- list(
        x0 = price_init,
        fn = profit_partial,
        lower = bounds$lower,
        upper = bounds$upper,
        hin = constraint_market_share,
        nl.info = FALSE,
        control = list(maxeval = 1000, xtol_rel = 1e-6)
      )
      if ("deprecatedBehavior" %in% names(formals(nloptr::slsqp))) {
        slsqp_args$deprecatedBehavior <- TRUE
      }

      res <- tryCatch(
        do.call(nloptr::slsqp, slsqp_args),
        error = function(e) {
          warning(sprintf("Optimization failed for insurer %s at t=%s: %s", name, t, e$message))
          NULL
        }
      )

      if (!is.null(res) && !is.null(res$par) && isTRUE(res$convergence >= 0)) {
        price_obj <- archetype_vector_to_object(res$par, current_archetype[[name]])
        prices_std_new[[name]] <- reshape_price_to_standard(price_obj, current_archetype[[name]])
      }
    }

    if (exists("DAMPING_NU")) {
      for (nm in names(prices_std_new)) {
        prices_std_new[[nm]] <- (1 - DAMPING_NU) * prices_std_prev[[nm]] + DAMPING_NU * prices_std_new[[nm]]
      }
    }

    for (name in names(ASSURERS_CONFIG)) {
      prices_std_history[[name]][[length(prices_std_history[[name]]) + 1]] <- prices_std_new[[name]]
      price_archetype_form <- reshape_price_from_standard(prices_std_new[[name]], current_archetype[[name]])
      prices_opts[[name]][[length(prices_opts[[name]]) + 1]] <- price_archetype_form
    }
  }

  records <- list()
  for (t in 0:T_MAX) {
    prices_for_shares <- list()
    for (name in sort(names(ASSURERS_CONFIG))) {
      prices_for_shares[[name]] <- prices_std_history[[name]][[t + 1]]
    }
    shares_dict <- shares_infr_comm(prices_for_shares)

    pure_ir <- outer(u, risk_cost * risk_prob)
    lr_comm <- list()
    lr_global <- list()
    for (assurer_name in sort(names(ASSURERS_CONFIG))) {
      share_ir <- shares_dict[[assurer_name]]
      prix_ir <- prices_for_shares[[assurer_name]]
      weight_ir <- share_ir * matrix(N, nrow = I, ncol = 3) * risk_props
      claims_comm <- rowSums(weight_ir * pure_ir)
      premiums_comm <- rowSums(weight_ir * prix_ir)
      lr_comm[[assurer_name]] <- ifelse(premiums_comm > 0, claims_comm / premiums_comm, NA_real_)
      claims_global <- sum(weight_ir * pure_ir)
      premiums_global <- sum(weight_ir * prix_ir)
      lr_global[[assurer_name]] <- ifelse(premiums_global > 0, claims_global / premiums_global, NA_real_)
    }

    alpha_u <- matrix(alpha * u, nrow = I, ncol = 3)
    utilities <- list()
    for (assurer_name in sort(names(ASSURERS_CONFIG))) {
      utilities[[assurer_name]] <- alpha_u + get_v_insurer(assurer_name) - BETA_PRICE * prices_for_shares[[assurer_name]]
    }
    exp_util_sum <- Reduce("+", lapply(utilities, function(mat) exp(pmin(pmax(mat, -500.0), 400.0))))
    logsum_ir <- (1 / BETA_PRICE) * log(exp_util_sum)

    N_mat <- matrix(N, nrow = I, ncol = 3)
    welfare_ea_comm <- rowSums(logsum_ir * risk_props) * N
    welfare_ea_risk <- colSums(logsum_ir * risk_props * N_mat)
    welfare_ea_global <- sum(welfare_ea_comm)
    welfare_ea_ir <- logsum_ir * risk_props * N_mat

    avg_util_ir <- matrix(0, nrow = I, ncol = 3)
    for (assurer_name in sort(names(ASSURERS_CONFIG))) {
      share_ir <- shares_dict[[assurer_name]]
      prix_ir <- prices_for_shares[[assurer_name]]
      avg_util_ir <- avg_util_ir + share_ir * (pure_ir - prix_ir)
    }
    welfare_ep_ir <- N_mat * risk_props * avg_util_ir
    welfare_ep_comm <- rowSums(welfare_ep_ir)
    welfare_ep_risk <- colSums(welfare_ep_ir)
    welfare_ep_global <- sum(welfare_ep_ir)

    for (i in seq_len(I)) {
      for (r in 1:3) {
        row_base <- list(
          period = t,
          commune_id = i - 1,
          risk_class = r - 1,
          commune_name = sprintf("commune_%s", i - 1),
          N_commune = N[i],
          risk_proportion = risk_props[i, r],
          risk_cost = risk_cost[r],
          risk_prob = u[i]
        )

        for (assurer_name in sort(names(ASSURERS_CONFIG))) {
          prix <- prices_for_shares[[assurer_name]][i, r]
          prixpure <- risk_cost[r] * risk_prob[r] * u[i]
          row_base[[sprintf("%s_prix", assurer_name)]] <- prix
          row_base[[sprintf("%s_prixpure", assurer_name)]] <- prixpure
          row_base[[sprintf("%s_marge_infr", assurer_name)]] <- prix - prixpure
        }

        for (assurer_name in sort(names(ASSURERS_CONFIG))) {
          share_val <- shares_dict[[assurer_name]][i, r]
          prix <- prices_for_shares[[assurer_name]][i, r]
          prixpure <- risk_cost[r] * risk_prob[r] * u[i]
          row_base[[sprintf("%s_part", assurer_name)]] <- share_val
          row_base[[sprintf("%s_profit_contrib", assurer_name)]] <-
            share_val * N[i] * risk_props[i, r] * (prix - prixpure)
          row_base[[sprintf("%s_LR_ir", assurer_name)]] <- ifelse(prix > 0, prixpure / prix, NA_real_)
          row_base[[sprintf("%s_LR_comm", assurer_name)]] <- lr_comm[[assurer_name]][i]
          row_base[[sprintf("%s_LR_global", assurer_name)]] <- lr_global[[assurer_name]]
        }

        row_base$welfare_ea_ir <- welfare_ea_ir[i, r]
        row_base$welfare_ea_comm <- welfare_ea_comm[i]
        row_base$welfare_ea_risk <- welfare_ea_risk[r]
        row_base$welfare_ea_global <- welfare_ea_global
        row_base$welfare_ep_ir <- welfare_ep_ir[i, r]
        row_base$welfare_ep_comm <- welfare_ep_comm[i]
        row_base$welfare_ep_risk <- welfare_ep_risk[r]
        row_base$welfare_ep_global <- welfare_ep_global

        records[[length(records) + 1]] <- row_base
      }
    }
  }

  out <- do.call(rbind.data.frame, lapply(records, as.data.frame))
  out$scenario_key <- scenario
  out
}
```

# Generate the four central scenarios

``` r
scenario_specs <- list(
  central_unconstrained = list(label = "Unconstrained", scenario = "unconstrained"),
  central_weak_global   = list(label = "National constraint", scenario = "national"),
  central_strong_local  = list(label = "Commune-level constraint", scenario = "commune"),
  central_risk_class    = list(label = "Risk-class constraint", scenario = "risk_class")
)

scenario_data <- list()
for (nm in names(scenario_specs)) {
  spec <- scenario_specs[[nm]]
  message("Running ", spec$label)
  scenario_data[[spec$label]] <- run_market_simulation(
    scenario = spec$scenario,
    seed = params$seed,
    I = params$n_communes,
    N_POP = params$n_pop,
    MS_CONSTRAINT = params$ms_constraint,
    T_MAX = params$t_max,
    T_INSPECT = params$t_inspect
  )
  if (isTRUE(params$export_csv)) {
    write.csv(
      scenario_data[[spec$label]],
      file.path(params$csv_dir, paste0(nm, ".csv")),
      row.names = FALSE
    )
  }
}
```

# Figure helpers

``` r
scenario_levels <- c(
  "Unconstrained",
  "National constraint",
  "Commune-level constraint",
  "Risk-class constraint"
)

risk_levels <- c(
  "Low risk (r = 1)",
  "Moderate risk (r = 2)",
  "High risk (r = 3)"
)

risk_label_clean <- function(x) {
  out <- dplyr::case_when(
    as.character(x) %in% c("0", "1") ~ "Low risk (r = 1)",
    as.character(x) %in% c("1", "2") & as.numeric(as.character(x)) == 1 ~ "Moderate risk (r = 2)",
    as.character(x) %in% c("2", "3") & as.numeric(as.character(x)) == 2 ~ "High risk (r = 3)",
    TRUE ~ as.character(x)
  )
  # Safer explicit mapping for the simulation output, where risk_class is 0, 1, 2.
  out[as.character(x) == "0"] <- "Low risk (r = 1)"
  out[as.character(x) == "1"] <- "Moderate risk (r = 2)"
  out[as.character(x) == "2"] <- "High risk (r = 3)"
  factor(out, levels = risk_levels)
}

insurer_palette <- c(A = "#A6A6A6", B = "#1F77B4", C = "#D62728")
risk_palette <- c(
  "Low risk (r = 1)" = "#2ECC71",
  "Moderate risk (r = 2)" = "#F1C40F",
  "High risk (r = 3)" = "#EF3B2C"
)

weighted_mean_safe <- function(x, w) {
  ok <- is.finite(x) & is.finite(w) & w > 0
  if (!any(ok)) return(NA_real_)
  sum(x[ok] * w[ok]) / sum(w[ok])
}

make_scenario_bind <- function(fun) {
  bind_rows(lapply(names(scenario_data), function(sc) {
    fun(scenario_data[[sc]], sc)
  })) %>%
    mutate(scenario = factor(scenario, levels = scenario_levels))
}

save_plot <- function(plot, filename, width, height, dpi = 300) {
  png_path <- file.path(params$fig_dir, paste0(filename, ".png"))
  ggsave(png_path, plot = plot, width = width, height = height, dpi = dpi)
  if (isTRUE(params$save_pdf)) {
    pdf_path <- file.path(params$fig_dir, paste0(filename, ".pdf"))
    ggsave(pdf_path, plot = plot, width = width, height = height)
  }
  invisible(png_path)
}
```

# Figure 1: relative premiums

``` r
make_premium_by_risk_all_insurers <- function(df, scenario, n_periods = 24) {
  df %>%
    filter(period <= n_periods) %>%
    mutate(
      risk_label = risk_label_clean(risk_class),
      weight = N_commune * risk_proportion
    ) %>%
    pivot_longer(
      cols = all_of(c("A_prix", "B_prix", "C_prix")),
      names_to = "insurer",
      names_pattern = "(.*)_prix",
      values_to = "price"
    ) %>%
    group_by(insurer, period, risk_label) %>%
    summarise(price = weighted_mean_safe(price, weight), .groups = "drop") %>%
    arrange(insurer, risk_label, period) %>%
    group_by(insurer, risk_label) %>%
    mutate(price_index = price / first(price)) %>%
    ungroup() %>%
    mutate(
      scenario = scenario,
      insurer = factor(insurer, levels = c("A", "B", "C")),
      risk_label = factor(risk_label, levels = risk_levels)
    )
}

premium_all <- make_scenario_bind(make_premium_by_risk_all_insurers)
```

``` r
p_premium <- ggplot(
  premium_all,
  aes(x = period, y = price_index, color = risk_label)
) +
  geom_hline(yintercept = 1, linewidth = 0.35, color = "grey35") +
  geom_line(linewidth = 1.0) +
  facet_grid(
    rows = vars(insurer),
    cols = vars(scenario),
    labeller = labeller(insurer = c(A = "Insurer A", B = "Insurer B", C = "Insurer C"))
  ) +
  scale_color_manual(values = risk_palette, drop = FALSE) +
  labs(
    title = "Relative premium paths by insurer and scenario",
    x = "Time (periods)",
    y = "Price index (base 1 = initial price)",
    color = NULL
  ) +
  theme_minimal(base_size = 12) +
  theme(
    legend.position = "bottom",
    strip.text = element_text(face = "bold")
  )

p_premium
```

<img src="figures/premium-facets-all-insurers-1.png" style="display: block; margin: auto;" />

``` r
save_plot(p_premium, "premium-facets-all-insurers", width = 13, height = 10)
```

# Figure 2: market shares by risk class

``` r
make_market_share_by_risk <- function(df, scenario, n_periods = 10) {
  df %>%
    filter(period <= n_periods) %>%
    mutate(
      risk_label = risk_label_clean(risk_class),
      weight = N_commune * risk_proportion
    ) %>%
    pivot_longer(
      cols = all_of(c("A_part", "B_part", "C_part")),
      names_to = "insurer",
      names_pattern = "(.*)_part",
      values_to = "share"
    ) %>%
    group_by(period, risk_label, insurer) %>%
    summarise(avg_share = weighted_mean_safe(share, weight), .groups = "drop") %>%
    mutate(
      scenario = scenario,
      scenario = factor(scenario, levels = scenario_levels),
      risk_label = factor(risk_label, levels = risk_levels),
      insurer = factor(insurer, levels = c("A", "B", "C"))
    )
}

ms_all <- make_scenario_bind(make_market_share_by_risk)
```

``` r
p_market <- ggplot(ms_all, aes(x = factor(period), y = avg_share, fill = insurer)) +
  geom_col(width = 0.85) +
  facet_grid(rows = vars(risk_label), cols = vars(scenario), drop = FALSE) +
  scale_y_continuous(
    labels = percent_format(accuracy = 1),
    breaks = seq(0, 1, by = 0.25),
    limits = c(0, 1),
    expand = expansion(mult = c(0, 0.02))
  ) +
  scale_fill_manual(values = insurer_palette, drop = FALSE) +
  labs(
    title = "Market shares by risk class and scenario",
    x = "Time (periods)",
    y = "Market share",
    fill = NULL
  ) +
  theme_minimal(base_size = 11) +
  theme(
    legend.position = "bottom",
    strip.text = element_text(face = "bold"),
    axis.text.x = element_text(angle = 45, hjust = 1),
    panel.grid.major.x = element_blank()
  )

p_market
```

<img src="figures/market-share-facets-1.png" style="display: block; margin: auto;" />

``` r
save_plot(p_market, "market-share-facets", width = 13, height = 8)
```

# Figure 3: portfolio composition

``` r
make_portfolio_composition <- function(df, scenario, n_periods = 10) {
  df %>%
    filter(period <= n_periods) %>%
    mutate(
      risk_label = risk_label_clean(risk_class),
      segment_weight = N_commune * risk_proportion
    ) %>%
    pivot_longer(
      cols = all_of(c("A_part", "B_part", "C_part")),
      names_to = "insurer",
      names_pattern = "(.*)_part",
      values_to = "segment_market_share"
    ) %>%
    mutate(portfolio_mass = segment_weight * segment_market_share) %>%
    group_by(period, insurer, risk_label) %>%
    summarise(portfolio_mass = sum(portfolio_mass, na.rm = TRUE), .groups = "drop") %>%
    tidyr::complete(
      period,
      insurer = c("A", "B", "C"),
      risk_label = factor(risk_levels, levels = risk_levels),
      fill = list(portfolio_mass = 0)
    ) %>%
    mutate(
      scenario = scenario,
      scenario = factor(scenario, levels = scenario_levels),
      insurer = factor(insurer, levels = c("A", "B", "C")),
      risk_label = factor(risk_label, levels = risk_levels)
    )
}

portfolio_all <- make_scenario_bind(make_portfolio_composition)
```

``` r
p_portfolio <- ggplot(
  portfolio_all,
  aes(x = factor(period), y = portfolio_mass, fill = risk_label)
) +
  geom_col(width = 0.85, position = "fill") +
  facet_grid(
    rows = vars(insurer),
    cols = vars(scenario),
    labeller = labeller(insurer = c(A = "Insurer A", B = "Insurer B", C = "Insurer C")),
    drop = FALSE
  ) +
  scale_y_continuous(
    labels = percent_format(accuracy = 1),
    breaks = seq(0, 1, by = 0.25),
    expand = expansion(mult = c(0, 0.02))
  ) +
  scale_fill_manual(values = risk_palette, drop = FALSE) +
  labs(
    title = "Portfolio composition by insurer and scenario",
    x = "Time (periods)",
    y = "Portfolio composition",
    fill = NULL
  ) +
  theme_minimal(base_size = 11) +
  theme(
    legend.position = "bottom",
    strip.text = element_text(face = "bold"),
    axis.text.x = element_text(angle = 45, hjust = 1),
    panel.grid.major.x = element_blank()
  )

p_portfolio
```

<img src="figures/portfolio-composition-facets-1.png" style="display: block; margin: auto;" />

``` r
save_plot(p_portfolio, "portfolio-composition-facets", width = 13, height = 8)
```

# Figure 4: loss ratios

``` r
compute_claims_premiums_long <- function(df, insurers = c("A", "B", "C")) {
  bind_rows(lapply(insurers, function(ins) {
    df %>%
      transmute(
        period,
        commune_id,
        risk_class,
        insurer = ins,
        weight = .data[[paste0(ins, "_part")]] * N_commune * risk_proportion,
        claims = weight * .data[[paste0(ins, "_prixpure")]],
        premiums = weight * .data[[paste0(ins, "_prix")]]
      )
  }))
}

make_loss_ratio_by_risk <- function(df, scenario, n_periods = 24) {
  compute_claims_premiums_long(df, insurers = c("A", "B", "C")) %>%
    filter(period <= n_periods) %>%
    group_by(period, risk_class, insurer) %>%
    summarise(
      claims = sum(claims, na.rm = TRUE),
      premiums = sum(premiums, na.rm = TRUE),
      .groups = "drop"
    ) %>%
    mutate(
      loss_ratio = claims / premiums,
      risk_label = risk_label_clean(risk_class),
      scenario = scenario,
      scenario = factor(scenario, levels = scenario_levels),
      risk_label = factor(risk_label, levels = risk_levels),
      insurer = factor(insurer, levels = c("A", "B", "C"))
    )
}

lr_all <- make_scenario_bind(make_loss_ratio_by_risk)
```

``` r
p_lr <- ggplot(lr_all, aes(x = period, y = loss_ratio, color = insurer)) +
  annotate("rect", xmin = -Inf, xmax = Inf, ymin = -Inf, ymax = 1, alpha = 0.08, fill = "#B7E4C7") +
  annotate("rect", xmin = -Inf, xmax = Inf, ymin = 1, ymax = Inf, alpha = 0.08, fill = "#F7B6B2") +
  geom_hline(yintercept = 1, linewidth = 0.35, color = "black") +
  geom_line(linewidth = 0.95) +
  facet_grid(
    rows = vars(risk_label),
    cols = vars(scenario),
    scales = "free_y",
    drop = FALSE
  ) +
  scale_color_manual(values = insurer_palette, drop = FALSE) +
  labs(
    title = "Loss ratios by risk class and scenario",
    x = "Time (periods)",
    y = "Loss ratio (S/P)",
    color = NULL
  ) +
  theme_minimal(base_size = 11) +
  theme(
    legend.position = "bottom",
    strip.text = element_text(face = "bold")
  )

p_lr
```

<img src="figures/loss-ratio-facets-1.png" style="display: block; margin: auto;" />

``` r
save_plot(p_lr, "loss-ratio-facets", width = 13, height = 8)
```
