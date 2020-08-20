#### Porep-rust源码梳理
lotus与rust的交互逻辑在filecoin-ffi这个库中，通过cgo的方式调用rust语言

#### add Picece
入口文件: 在  filecoin-ffi/rust/src/proofs/api.rs  这个文件中

    #[no_mangle]
    #[cfg(not(target_os = "windows"))]
    pub unsafe extern "C" fn fil_write_with_alignment(
        registered_proof: fil_RegisteredSealProof,   // proof类型 32G等
        src_fd: libc::c_int,                         // 文件输入标识符
        src_size: u64,                               // 输入大小  
        dst_fd: libc::c_int,                         // 文件输入出标识符
        existing_piece_sizes_ptr: *const u64,        // 存在piece引用 
        existing_piece_sizes_len: libc::size_t,      // piece 长度
    ) -> *mut fil_WriteWithAlignmentResponse {
        ...
          match filecoin_proofs_api::seal::add_piece(
            registered_proof.into(),
            FileDescriptorRef::new(src_fd),
            FileDescriptorRef::new(dst_fd),
            n,
            &piece_sizes,
        ) {
        ...
    }


    ...
在 filecoin_proofs_api/src/seal.rs  文件中
   
    pub fn add_piece<R, W>(
    registered_proof: RegisteredSealProof,
    source: R,
    target: W,
    piece_size: UnpaddedBytesAmount,
    piece_lengths: &[UnpaddedBytesAmount],
    ) -> Result<(PieceInfo, UnpaddedBytesAmount)>
    where
        R: Read,
        W: Read + Write + Seek,
    {
        use RegisteredSealProof::*;
        match registered_proof {
            StackedDrg2KiBV1 | StackedDrg8MiBV1 | StackedDrg512MiBV1 | StackedDrg32GiBV1
            | StackedDrg64GiBV1 => {
                filecoin_proofs_v1::add_piece(source, target, piece_size, piece_lengths)
            }
        }
    }

在 filecoin_proffs/src/api/mod.rs 文件中

    pub fn add_piece<R, W>(
    source: R,
    target: W,
    piece_size: UnpaddedBytesAmount,
    piece_lengths: &[UnpaddedBytesAmount],
    ) -> Result<(PieceInfo, UnpaddedBytesAmount)>
    where
        R: Read,
        W: Write,
    {
        ...
        // todo 不能直接生成一个文件重复使用？
         for _ in 0..usize::from(PaddedBytesAmount::from(piece_alignment.left_bytes)) {
            target.write_all(&[0u8][..])?; // 写入 0
        }
        ...
    }

#### P1

filecoin-ffi/rust/src/proofs/api.rs

    #[no_mangle]
    pub unsafe extern "C" fn fil_seal_pre_commit_phase1(
        registered_proof: fil_RegisteredSealProof,  // 证明类型
        cache_dir_path: *const libc::c_char,        // 缓存目录
        staged_sector_path: *const libc::c_char,    // 暂存路径
        sealed_sector_path: *const libc::c_char,    // sealed路径
        sector_id: u64,                             // 扇区id
        prover_id: fil_32ByteArray,                 // 生命id
        ticket: fil_32ByteArray,                    // 随机数
        pieces_ptr: *const fil_PublicPieceInfo,     // pieces 地址 
        pieces_len: libc::size_t,                   // pieces 长度
    ) -> *mut fil_SealPreCommitPhase1Response {
        ...
        let result = filecoin_proofs_api::seal::seal_pre_commit_phase1(
            registered_proof.into(),
            c_str_to_pbuf(cache_dir_path),
            c_str_to_pbuf(staged_sector_path),
            c_str_to_pbuf(sealed_sector_path),
            prover_id.inner,
            SectorId::from(sector_id),
            ticket.inner,
            &public_pieces,
        )
        ...
    }

 filecoin_proofs_api/src/seal.rs    

    ...
    let output = filecoin_proofs_v1::seal_pre_commit_phase1::<_, _, _, Tree>(
        config,
        cache_path,
        in_path,              // staged 文件路径
        out_path,             // 输出路径
        prover_id,
        sector_id,
        ticket,
        piece_infos,
    )?;
    ...

filecoin-proofs/src/api/seal.rs

    ...
    pub fn seal_pre_commit_phase1<R, S, T, Tree: 'static + MerkleTreeTrait>(
        porep_config: PoRepConfig,
        cache_path: R,
        in_path: S,
        out_path: T,
        prover_id: ProverId,
        sector_id: SectorId,
        ticket: Ticket,
        piece_infos: &[PieceInfo],
    ) -> Result<SealPreCommitPhase1Output<Tree>>{
    ...
    // todo 把 unsealed 数据拷贝到 sealed的位置处，此处可优化直接，直接把unsealed数据放到 sealed位置处，节省拷贝时间
     fs::copy(&in_path, &out_path).with_context(|| {
        format!(
            "could not copy in_path={:?} to out_path={:?}",
            in_path.as_ref().display(),
            out_path.as_ref().display()
        )
    })?;
    ...
    // 前面的操作把扇区进行了内存文件映射，计算默克尔数信息相关东西 ，开始计算11层的 SDRG
    let labels = StackedDrg::<Tree, DefaultPieceHasher>::replicate_phase1(
        &compound_public_params.vanilla_params,
        &replica_id,
        config.clone(),
    )?;
}

storage-proofs-porep/src/stacked/vanilla/proofs.rs

    // 开始计算生成11层
    fn generate_labels(
        graph: &StackedBucketGraph<Tree::Hasher>,
        layer_challenges: &LayerChallenges,
        replica_id: &<Tree::Hasher as Hasher>::Domain,
        config: StoreConfig,
    ) -> Result<(LabelsCache<Tree>, Labels<Tree>)> {
        ...
        // SDRG的层数
        let layers = layer_challenges.layers();
        // 存储每一层文件配置信息
        let mut labels: Vec<DiskStore<<Tree::Hasher as Hasher>::Domain>> =
            Vec::with_capacity(layers);
        // 每一层配置信息    
        let mut label_configs: Vec<StoreConfig> = Vec::with_capacity(layers);

        // 每一层计算SDRG node 的数量 ? 
        let layer_size = graph.size() * NODE_SIZE;
        // 缓存当前层，下一层需要用到上一层的信息
        let mut layer_labels = vec![0u8; layer_size]; // Buffer for labels of the current layer
        // 缓存下一层，
        let mut exp_labels = vec![0u8; layer_size]; // Buffer for labels of the previous layer, needed for expander parents

        let use_cache = settings::SETTINGS.lock().unwrap().maximize_caching;
        // 是否使用缓存，默认的使用，大概57的大小，这个可配置，可以优化的从ssd读取?
        let mut cache = if use_cache {
            Some(graph.parent_cache()?)
        } else {
            None
        };
        for layer in 1..=layers {
            info!("generating layer: {}", layer);
            if let Some(ref mut cache) = cache {
                cache.reset()?;
            }

            if layer == 1 {
                // 第一层，
                for node in 0..graph.size() {
                    create_label(
                        graph,
                        cache.as_mut(),
                        replica_id,
                        &mut layer_labels,
                        layer,
                        node,
                    )?;
                }
            } else {
                for node in 0..graph.size() {
                    create_label_exp(
                        graph,
                        cache.as_mut(),
                        replica_id,
                        &exp_labels,
                        &mut layer_labels,
                        layer,
                        node,
                    )?;
                }
            }

            // Write the result to disk to avoid keeping it in memory all the time.
            let layer_config =
                StoreConfig::from_config(&config, CacheKey::label_layer(layer), Some(graph.size()));

            info!("  storing labels on disk");
            // Construct and persist the layer data.
            let layer_store: DiskStore<<Tree::Hasher as Hasher>::Domain> =
                DiskStore::new_from_slice_with_config(
                    graph.size(),
                    Tree::Arity::to_usize(),
                    &layer_labels,
                    layer_config.clone(),
                )?;
            info!(
                "  generated layer {} store with id {}",
                layer, layer_config.id
            );

            info!("  setting exp parents");
            // 将最新一层的数据交换，存储在内存中 ，两层来算就是64G， 这个可以优化从ssd上拉取数据 ?
            std::mem::swap(&mut layer_labels, &mut exp_labels);

            // Track the layer specific store and StoreConfig for later retrieval.
            labels.push(layer_store);
            label_configs.push(layer_config);

    }













