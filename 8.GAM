options(encoding = "UTF-8")

suppressPackageStartupMessages({
  library(data.table)
  library(mgcv)
  library(ggplot2)
})

mode_levels <- c("Bus", "Bike", "Subway", "Taxi")

mode_map <- data.table(
  mode = mode_levels,
  y = 0:3,
  color = c("#F1C40F", "#2CA25F", "#D73027", "#2B6CB0")
)

default_od_pairs <- c(
  "3->3", "4->4", "1->1", "0->0",
  "3->4", "4->3", "3->0", "0->3",
  "3->1", "1->3", "0->4", "4->0"
)

od_labels <- c(
  "0" = "CRC",
  "1" = "ILS",
  "3" = "CRS",
  "4" = "BFS"
)

period_order <- c("morning_peak", "midday", "evening_peak", "night")

period_labels <- c(
  morning_peak = "Morning peak",
  midday = "Midday",
  evening_peak = "Evening peak",
  night = "Night"
)

period_time_points <- list(
  morning_peak = c(7, 7.5, 8, 8.5),
  midday = c(11, 11.5, 12, 12.5, 13, 13.5),
  evening_peak = c(17, 17.5, 18, 18.5, 19, 19.5),
  night = c(0, 0.5, 1, 1.5, 2, 2.5, 3, 3.5, 4, 4.5, 20, 20.5, 21, 21.5, 22, 22.5, 23, 23.5)
)

od_label <- function(od_pair) {
  parts <- strsplit(od_pair, "->", fixed = TRUE)[[1]]
  if (length(parts) != 2) return(od_pair)
  left <- od_labels[parts[1]]
  right <- od_labels[parts[2]]
  if (is.na(left) || is.na(right)) return(od_pair)
  paste0(left, "-", right)
}

aggregate_interaction_counts <- function(trips,
                                         distance_bin_width = 0.25,
                                         time_bin_width = 0.25) {
  dt <- as.data.table(trips)
  required <- c("od_pair", "mode", "distance_km", "time_hour")
  missing <- setdiff(required, names(dt))
  if (length(missing)) stop("Missing required columns: ", paste(missing, collapse = ", "))

  dt <- merge(dt, mode_map[, .(mode, y)], by = "mode", all.x = TRUE, sort = FALSE)
  dt <- dt[!is.na(y) & is.finite(distance_km) & distance_km > 0 & is.finite(time_hour)]
  dt[, `:=`(
    time_hour = time_hour %% 24,
    distance_bin = floor(distance_km / distance_bin_width) * distance_bin_width + distance_bin_width / 2,
    time_bin = floor(time_hour / time_bin_width) * time_bin_width + time_bin_width / 2
  )]
  dt[time_bin >= 24, time_bin := time_bin - 24]
  dt[, log_distance := log(pmax(distance_bin, distance_bin_width / 4))]
  dt[, .(count = .N), by = .(od_pair, y, mode, distance_bin, time_bin, log_distance)]
}

complete_grid_counts <- function(counts) {
  dt <- as.data.table(counts)
  base <- CJ(
    od_pair = sort(unique(dt$od_pair)),
    distance_bin = sort(unique(dt$distance_bin)),
    time_bin = sort(unique(dt$time_bin)),
    sorted = FALSE
  )
  base[, log_distance := log(pmax(distance_bin, 0.025))]
  base[, join_key := 1L]
  modes <- copy(mode_map[, .(y, mode)])
  modes[, join_key := 1L]
  grid <- merge(base, modes, by = "join_key", allow.cartesian = TRUE)
  grid[, join_key := NULL]
  out <- merge(
    grid,
    dt,
    by = c("od_pair", "distance_bin", "time_bin", "log_distance", "y", "mode"),
    all.x = TRUE,
    sort = FALSE
  )
  out[is.na(count), count := 0L]
  out[, y := as.integer(y)]
  out[]
}

fit_interaction_gam <- function(counts, kd = 8L, kt = 9L) {
  dt <- as.data.table(counts)[count > 0]
  if (length(unique(dt$y)) < length(mode_levels)) {
    stop("The model requires all four modes in the fitting data.")
  }

  rhs <- paste0(
    "s(log_distance,k=", kd, ") + ",
    "s(time_bin,bs='cc',k=", kt, ") + ",
    "te(log_distance,time_bin,k=c(", kd, ",", kt, "),bs=c('tp','cc'))"
  )
  forms <- list(
    as.formula(paste("y ~", rhs)),
    as.formula(paste("~", rhs)),
    as.formula(paste("~", rhs))
  )

  gam(
    forms,
    family = multinom(K = length(mode_levels) - 1L),
    data = dt,
    weights = count,
    method = "REML",
    knots = list(time_bin = c(0, 24))
  )
}

softmax_from_eta <- function(eta) {
  eta <- as.matrix(eta)
  ex <- exp(pmin(eta, 30))
  den <- 1 + rowSums(ex)
  out <- cbind(
    Bus = 1 / den,
    Bike = ex[, 1] / den,
    Subway = ex[, 2] / den,
    Taxi = ex[, 3] / den
  )
  colnames(out) <- mode_levels
  out
}

distance_prediction_grid <- function(counts, n = 420L) {
  dt <- as.data.table(counts)[count > 0]
  x <- seq(min(dt$distance_bin, na.rm = TRUE), max(dt$distance_bin, na.rm = TRUE), length.out = n)
  data.table(distance_bin = x, log_distance = log(pmax(x, 0.025)))
}

period_curve_grid <- function(counts, n_distance = 420L) {
  d <- distance_prediction_grid(counts, n = n_distance)
  rbindlist(lapply(names(period_time_points), function(p) {
    grid <- CJ(distance_bin = d$distance_bin, time_bin = period_time_points[[p]], sorted = FALSE)
    grid[, `:=`(log_distance = log(pmax(distance_bin, 0.025)), period = p)]
    grid[, .(distance_bin, log_distance, time_bin, period)]
  }))
}

predict_gam_probabilities <- function(model, newdata, sim_b = 200L, seed = 20260714L) {
  pred <- predict(model, newdata = newdata, type = "link", se.fit = TRUE)
  eta <- pred$fit
  prob <- softmax_from_eta(eta)
  out <- cbind(as.data.table(newdata), as.data.table(prob))
  long <- melt(out, measure.vars = mode_levels, variable.name = "mode", value.name = "p_fit")
  long[, mode := as.character(mode)]

  if (sim_b > 0) {
    se <- pmax(pred$se.fit, 1e-8)
    p_array <- array(NA_real_, dim = c(nrow(newdata), length(mode_levels), sim_b))
    set.seed(seed)
    for (b in seq_len(sim_b)) {
      eta_b <- matrix(
        rnorm(length(eta), mean = as.numeric(eta), sd = as.numeric(se)),
        nrow = nrow(eta),
        ncol = ncol(eta)
      )
      p_array[, , b] <- softmax_from_eta(eta_b)
    }
    for (m in mode_levels) {
      mi <- match(m, mode_levels)
      vals <- p_array[, mi, , drop = FALSE][, 1, ]
      p_low <- apply(vals, 1, quantile, probs = 0.025, na.rm = TRUE)
      p_high <- apply(vals, 1, quantile, probs = 0.975, na.rm = TRUE)
      long[mode == m, `:=`(
        p_low = p_low,
        p_high = p_high,
        confidence = pmax(0, pmin(1, 1 - (p_high - p_low)))
      )]
    }
  } else {
    long[, `:=`(p_low = NA_real_, p_high = NA_real_, confidence = NA_real_)]
  }
  long[]
}

predict_period_curves <- function(model, counts, n_distance = 420L, sim_b = 200L) {
  grid <- period_curve_grid(counts, n_distance = n_distance)
  pred <- predict_gam_probabilities(model, grid, sim_b = sim_b)
  pred[, .(
    p_fit = mean(p_fit, na.rm = TRUE),
    p_low = mean(p_low, na.rm = TRUE),
    p_high = mean(p_high, na.rm = TRUE),
    confidence = mean(confidence, na.rm = TRUE)
  ), by = .(period, distance_bin, mode)]
}

threshold_windows_from_flags <- function(x,
                                         flag_col,
                                         min_width = 0.5,
                                         max_width = 5,
                                         max_segments = 6L) {
  x <- copy(x)[order(distance_bin)]
  x[, flag := fifelse(is.na(get(flag_col)), FALSE, get(flag_col))]
  if (!any(x$flag)) return(data.table())

  rr <- rle(x$flag)
  ends <- cumsum(rr$lengths)
  starts <- ends - rr$lengths + 1L
  out <- list()

  for (i in seq_along(rr$values)) {
    if (!rr$values[i]) next
    idx <- starts[i]:ends[i]
    if (length(idx) < 2) next

    candidates <- list()
    for (a_pos in seq_along(idx)) {
      a <- idx[a_pos]
      later <- idx[idx > a]
      if (!length(later)) next
      widths <- x$distance_bin[later] - x$distance_bin[a]
      ok <- which(widths >= min_width & widths <= max_width)
      if (!length(ok)) next

      for (j in later[ok]) {
        width <- x$distance_bin[j] - x$distance_bin[a]
        delta_p <- abs(x$p_fit[j] - x$p_fit[a])
        avg_abs_slope <- delta_p / max(width, 1e-8)
        confidence <- mean(x$confidence[a:j], na.rm = TRUE)
        if (is.nan(confidence)) confidence <- NA_real_
        candidates[[length(candidates) + 1L]] <- data.table(
          start = x$distance_bin[a],
          end = x$distance_bin[j],
          width = width,
          confidence = confidence,
          delta_p = delta_p,
          avg_abs_slope = avg_abs_slope,
          score = delta_p * avg_abs_slope * fifelse(is.na(confidence), 1, confidence) * 100
        )
      }
    }
    if (length(candidates)) out[[length(out) + 1L]] <- rbindlist(candidates, fill = TRUE)
  }

  ans <- rbindlist(out, fill = TRUE)
  if (!nrow(ans)) return(ans)
  head(ans[order(-delta_p, -avg_abs_slope, -confidence, start)], max_segments)
}

extract_threshold_intervals <- function(period_curves,
                                        od_pair = NA_character_,
                                        min_width = 0.5,
                                        max_width = 5,
                                        min_probability_change = 0.05,
                                        max_intervals = 2L) {
  dt <- as.data.table(period_curves)
  out <- list()

  for (p in unique(dt$period)) {
    for (m in mode_levels) {
      x <- dt[period == p & mode == m][order(distance_bin)]
      if (nrow(x) < 3) next
      x[, derivative := c(NA_real_, diff(p_fit) / diff(distance_bin))]
      x[, rising_flag := derivative > 0]
      x[, falling_flag := derivative < 0]

      for (flag in c("rising_flag", "falling_flag")) {
        seg <- threshold_windows_from_flags(
          x,
          flag_col = flag,
          min_width = min_width,
          max_width = max_width,
          max_segments = max_intervals * 3L
        )
        if (!nrow(seg)) next
        seg <- seg[delta_p >= min_probability_change]
        if (!nrow(seg)) next
        seg <- head(seg[order(-delta_p, -avg_abs_slope, -confidence)], max_intervals)
        seg[, `:=`(
          od_pair = od_pair,
          od_label = if (!is.na(od_pair)) od_label(od_pair) else NA_character_,
          period = p,
          mode = m,
          interval_type = if (flag == "rising_flag") "rising threshold interval" else "falling threshold interval",
          interval = paste0("[", sprintf("%.1f", start), ", ", sprintf("%.1f", end), "] km")
        )]
        out[[length(out) + 1L]] <- seg
      }
    }
  }

  ans <- rbindlist(out, fill = TRUE)
  if (!nrow(ans)) {
    return(data.table(
      od_pair = character(), od_label = character(), period = character(),
      mode = character(), interval_type = character(), interval = character(),
      start = numeric(), end = numeric(), width = numeric(), confidence = numeric(),
      delta_p = numeric(), avg_abs_slope = numeric(), score = numeric()
    ))
  }
  ans[, `:=`(
    start = round(start, 3),
    end = round(end, 3),
    width = round(width, 3),
    confidence = round(confidence, 3),
    delta_p = round(delta_p, 3),
    avg_abs_slope = round(avg_abs_slope, 4),
    score = round(score, 3)
  )]
  setcolorder(ans, c(
    "od_pair", "od_label", "period", "mode", "interval_type", "interval",
    "start", "end", "width", "confidence", "delta_p", "avg_abs_slope", "score"
  ))
  ans[]
}

plot_figure8_curves <- function(period_curves,
                                od_pairs = default_od_pairs,
                                threshold_bands = data.table(),
                                output_png = "figure8_gam_probability_curves.png",
                                output_pdf = "figure8_gam_probability_curves.pdf",
                                width = 7.7,
                                height = 14.7,
                                dpi = 450) {
  dt <- as.data.table(period_curves)
  od_pairs <- od_pairs[od_pairs %in% unique(dt$od_pair)]
  if (!length(od_pairs)) od_pairs <- sort(unique(dt$od_pair))

  label_map <- setNames(
    paste0("(", letters[seq_along(od_pairs)], ") ", vapply(od_pairs, od_label, character(1))),
    od_pairs
  )
  dt <- copy(dt[od_pair %in% od_pairs])
  dt[, od_row := factor(label_map[od_pair], levels = unname(label_map))]
  dt[, period := factor(period, levels = period_order, labels = period_labels[period_order])]
  dt[, mode := factor(mode, levels = mode_levels)]

  mode_linetypes <- c(Bus = "solid", Bike = "dashed", Subway = "dotdash", Taxi = "longdash")
  mode_linewidths <- c(Bus = 0.34, Bike = 0.34, Subway = 0.36, Taxi = 0.36)

  p <- ggplot(dt, aes(x = distance_bin, y = p_fit, color = mode, linetype = mode, group = mode))

  if (nrow(threshold_bands)) {
    bands <- as.data.table(threshold_bands)
    bands <- bands[is.finite(xmin) & is.finite(xmax) & xmax > xmin]
    lines <- data.table(xintercept = sort(unique(c(bands$xmin, bands$xmax))))
    p <- p +
      geom_rect(
        data = bands,
        aes(xmin = xmin, xmax = xmax, ymin = -Inf, ymax = Inf, fill = label),
        inherit.aes = FALSE,
        alpha = 0.055,
        color = NA
      ) +
      geom_vline(
        data = lines,
        aes(xintercept = xintercept),
        inherit.aes = FALSE,
        linetype = "dashed",
        linewidth = 0.10,
        color = "#333333",
        alpha = 0.45
      ) +
      scale_fill_manual(values = setNames(bands$fill, bands$label), name = "Estimated modal transition thresholds")
  }

  p <- p +
    geom_line(aes(linewidth = mode), lineend = "round") +
    scale_color_manual(values = setNames(mode_map$color, mode_map$mode), name = "Mode") +
    scale_linetype_manual(values = mode_linetypes, name = "Mode") +
    scale_linewidth_manual(values = mode_linewidths, name = "Mode") +
    scale_y_continuous(limits = c(0, 1.02), breaks = c(0, 0.5, 1.0)) +
    facet_grid(od_row ~ period, switch = "y") +
    labs(x = "Distance (km)", y = "Predicted probability") +
    theme_bw(base_size = 8.8, base_family = "Times New Roman") +
    theme(
      panel.grid.minor = element_line(color = "#f0f0f0", linewidth = 0.14),
      panel.grid.major = element_line(color = "#e6e6e6", linewidth = 0.20),
      panel.spacing = unit(0.045, "in"),
      legend.position = "bottom",
      legend.box = "vertical",
      legend.key.width = unit(0.40, "in"),
      legend.text = element_text(size = 7.4),
      legend.title = element_text(size = 8.0),
      legend.margin = margin(0, 0, 0, 0),
      strip.background = element_rect(fill = "white", color = "#d9d9d9", linewidth = 0.25),
      strip.text.x = element_text(size = 10.5, face = "bold"),
      strip.text.y.left = element_text(size = 9.2, angle = 90, face = "bold"),
      axis.title.x = element_text(size = 10.6, margin = margin(t = 3)),
      axis.title.y = element_text(size = 9.8, margin = margin(r = 4)),
      axis.text.x = element_text(size = 8.1),
      axis.text.y = element_text(size = 8.1),
      plot.margin = margin(4, 6, 4, 7)
    ) +
    guides(
      color = guide_legend(nrow = 1),
      linetype = guide_legend(nrow = 1),
      linewidth = "none",
      fill = guide_legend(nrow = 1)
    )

  ggsave(output_png, p, width = width, height = height, dpi = dpi)
  pdf_device <- if (capabilities("cairo")) grDevices::cairo_pdf else grDevices::pdf
  ggsave(output_pdf, p, width = width, height = height, device = pdf_device)
  invisible(p)
}
