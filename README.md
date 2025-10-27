import React, { useCallback, useEffect, useRef, useState } from "react";

export default function ImageSlider({
  apiEndpoint = "/api/images",
  pageSize = 50,
  autoplay = true,
  autoplayDelay = 4000,
  showThumbnails = true,
}) {
  const [images, setImages] = useState([]);
  const [index, setIndex] = useState(0);
  const [loading, setLoading] = useState(true);
  const [page, setPage] = useState(1);
  const [hasMore, setHasMore] = useState(true);
  const [isPlaying, setIsPlaying] = useState(Boolean(autoplay));
  const timerRef = useRef(null);
  const imgRefs = useRef([]);
  const observer = useRef(null);
  const touchStartX = useRef(null);

  const fetchImages = useCallback(async (pageToFetch = 1) => {
    try {
      setLoading(true);
      const res = await fetch(`${apiEndpoint}?page=${pageToFetch}&limit=${pageSize}`);
      if (!res.ok) throw new Error("Failed to fetch images");
      const payload = await res.json();
      const fetched = Array.isArray(payload.images) ? payload.images : [];
      setImages((prev) => (pageToFetch === 1 ? fetched : prev.concat(fetched)));
      setHasMore(payload.page < payload.totalPages);
      setPage(payload.page);
    } catch (err) {
      console.error(err);
    } finally {
      setLoading(false);
    }
  }, [apiEndpoint, pageSize]);

  useEffect(() => {
    fetchImages(1);
    return () => {
      if (timerRef.current) clearInterval(timerRef.current);
      if (observer.current) observer.current.disconnect?.();
    };
  }, [fetchImages]);

  const startAutoplay = useCallback(() => {
    if (!autoplay || timerRef.current) return;
    timerRef.current = setInterval(() => {
      setIndex((i) => (images.length ? (i + 1) % images.length : 0));
    }, autoplayDelay);
    setIsPlaying(true);
  }, [autoplay, autoplayDelay, images.length]);

  const stopAutoplay = useCallback(() => {
    if (timerRef.current) {
      clearInterval(timerRef.current);
      timerRef.current = null;
    }
    setIsPlaying(false);
  }, []);

  useEffect(() => {
    if (images.length && autoplay) startAutoplay();
    return stopAutoplay;
  }, [images.length, autoplay, startAutoplay, stopAutoplay]);

  useEffect(() => {
    if (!images.length) return;
    const nextIndex = (index + 1) % images.length;
    const nextUrl = images[nextIndex]?.url;
    if (nextUrl) {
      const img = new Image();
      img.src = nextUrl;
    }
    if (hasMore && images.length - index < 5) {
      fetchImages(page + 1);
    }
  }, [index, images, hasMore, page, fetchImages]);

  useEffect(() => {
    const onKey = (e) => {
      if (e.key === "ArrowLeft") movePrev();
      else if (e.key === "ArrowRight") moveNext();
      else if (e.key === " ") {
        e.preventDefault();
        isPlaying ? stopAutoplay() : startAutoplay();
      }
    };
    window.addEventListener("keydown", onKey);
    return () => window.removeEventListener("keydown", onKey);
  }, [isPlaying, startAutoplay, stopAutoplay]);

  const moveNext = useCallback(() => {
    setIndex((i) => (images.length ? (i + 1) % images.length : 0));
  }, [images.length]);

  const movePrev = useCallback(() => {
    setIndex((i) => (images.length ? (i - 1 + images.length) % images.length : 0));
  }, [images.length]);

  const onTouchStart = (e) => {
    stopAutoplay();
    touchStartX.current = e.touches?.[0]?.clientX ?? null;
  };
  const onTouchEnd = (e) => {
    if (touchStartX.current == null) return;
    const dx = (e.changedTouches?.[0]?.clientX ?? 0) - touchStartX.current;
    const threshold = 50;
    if (dx > threshold) movePrev();
    else if (dx < -threshold) moveNext();
    touchStartX.current = null;
  };

  useEffect(() => {
    if (!("IntersectionObserver" in window)) return;
    observer.current = new IntersectionObserver(
      (entries) => {
        entries.forEach((entry) => {
          if (entry.isIntersecting) {
            const img = entry.target;
            const src = img.dataset.src;
            if (src) {
              img.src = src;
              img.removeAttribute("data-src");
            }
            observer.current.unobserve(img);
          }
        });
      },
      { root: null, threshold: 0.15 }
    );
    imgRefs.current.forEach((n) => {
      if (n) observer.current.observe(n);
    });
    return () => observer.current?.disconnect();
  }, [images]);

  const attachImgRef = (el, i) => {
    imgRefs.current[i] = el;
  };

  return (
    <div
      className="relative w-full max-w-4xl mx-auto overflow-hidden rounded-lg shadow-lg"
      onMouseEnter={stopAutoplay}
      onMouseLeave={startAutoplay}
      onTouchStart={onTouchStart}
      onTouchEnd={onTouchEnd}
    >
      <div className="flex items-center justify-center bg-gray-200 h-80">
        {loading && <p>Loading...</p>}
        {!loading && images.length > 0 && (
          <img
            ref={(el) => attachImgRef(el, index)}
            data-src={images[index]?.url}
            alt={images[index]?.alt || "slide"}
            className="object-contain w-full h-80 transition-opacity duration-700 ease-in-out"
          />
        )}
      </div>

      <button
        className="absolute top-1/2 left-3 transform -translate-y-1/2 bg-black bg-opacity-40 text-white px-3 py-2 rounded-full"
        onClick={movePrev}
      >
        ‹
      </button>
      <button
        className="absolute top-1/2 right-3 transform -translate-y-1/2 bg-black bg-opacity-40 text-white px-3 py-2 rounded-full"
        onClick={moveNext}
      >
        ›
      </button>

      {showThumbnails && (
        <div className="flex justify-center mt-2 space-x-2 overflow-x-auto">
          {images.map((img, i) => (
            <img
              key={img.id || i}
              src={img.url}
              alt={img.alt || ""}
              onClick={() => setIndex(i)}
              className={`h-12 w-16 object-cover rounded cursor-pointer ${
                i === index ? "ring-2 ring-blue-500" : ""
              }`}
            />
          ))}
        </div>
      )}
    </div>
  );
}
